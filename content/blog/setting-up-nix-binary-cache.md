+++
title = "Setting Up a Nix Binary Cache"
date = 2020-08-26
[taxonomies]
tags = ["web"]
categories = ["devops"]
[extra]
discussion_id = "private-nix-binary-cache"
+++

This post builds on fzakaria's [article on setting up a binary cache on S3](https://fzakaria.com/2020/07/15/setting-up-a-nix-s3-binary-cache.html).


### Intro

On CI (e.g. Travis), where each build is run on a pristine environment, I need to be able to cache packages across builds to minimise build time. Also, for closed-source code, I need this cache to be private.

It is also possible to share binary cache within a development team, but I am not going to cover that here. There's a good [article](https://www.tweag.io/blog/2019-11-21-untrusted-ci/) by Tweag on that. This article is mainly aimed at solo developers or small teams where everyone can be trusted to push to the cache.


### Before we start

Make sure you have up-to-date version of nix. Older version of nix doesn't work properly with private S3 buckets. The version I'm using in this article is v2.3.7

```
$ nix --version
nix (Nix) 2.3.7
```


## Step by step guide

I'm going to walk through the process of setting up and using the cache, each time demonstrating 

0. Setup S3 bucket
1. Setup keys
2. Build & sign packages
3. Upload packages to S3
4. Use the cache as a substituter
5. Optimisation


### 0. Setup a S3 bucket

In this article, I have created a bucket named `fmnxl-nix-cache`.

#### Choosing a region

If you don't use the `us-east-1` region for your bucket, you will have to add the region to the S3 URI when you use it, e.g. `s3://fmnxl-nix-cache?region=eu-west-1`


### 1. Generate public and private keys

We need to generate a public and secret key pair in order to use binary caches. The secret key is used to sign paths before they're uploaded to the cache, and the public key will be used to verify that the paths downloaded are legitimate.

```
nix-store --generate-binary-cache-key \
  fmnxl-nix-cache \
  public.key \
  secret.key
```

I've named this pair of keys `fmnxl-nix-cache`. You can choose any name, but according to the manual:

> A key name, such as cache.example.org-1, that is used to look up keys on the client when it verifies signatures. It can be anything, but it's suggested to use the host name of your cache (e.g.  cache.example.org) with a suffix denoting the number of the key (to be incremented every time you need to revoke a key).

Now let's check our keys:

```
$ cat public.key 
fmnxl-nix-cache:LDCyJEmI12vrhpnUtS67eoc5ZfpK6wE91kENkdna04KlR9IGa9ggGNIX+FQJy2xisr49K7gL3VyKgZnXvTKjxw==c

$ cat secret.key 
fmnxl-nix-cache:Rm9OIEAAx3pjlnUNfdW2d/6pOWxlOBTrDrHyf9HB538=
```

### 2. Build and sign packages

Build your package
```
$ nix-build default.nix
/nix/store/xxxxxxxxxxxxxxxx-test
```

Then sign the resulting path recursively, so that dependencies are also included.
```
$ nix sign-paths \
  --recursive \
  --key-file my-key.sec \
  /nix/store/xxxxxxxxxxxxxxxx-test
```

You can check that your path has been correctly signed by inspecting it with `nix path-info`. You should be able to see that the path is signed with your secret key.

```
$ nix path-info --sigs /nix/store/xxxxxxxxxxxxxxxx-test
/nix/store/xxxxxxxxxxxxxxxx-fmnxl-test	fmnxl-nix-cache:XxXxXXXxxXxXX/XXXxXXXXxXXX/XXxxxxXX==
```

This can be skipped if you have set `secret-key-files` in `nix.conf`. See Optimisation section below.


### 3. Upload packages to the binary cache on S3

```
$ nix copy ---to s3://fmnxl-nix-cache /nix/store/xxxxxxxxxxxxxxxx-test
```

If your bucket is private, you would need to [setup your AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) on your machine. This can be in the form of setting environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

To check whether the path you've just uploaded has been properly signed, we can pass `--store s3://` to `nix path-info`

```
nix path-info --store s3://fmnxl-nix-cache --sigs /nix/store/xxxxxxxxxxxxxxxx-test
```

You should get the same output as when checking signatures on your local nix store. If not, see Troubleshooting section below.


### 4. Use the cache as a substituter

To see this in action we must first remove the package from the local `/nix/store`:

```
$ nix-store --delete /nix/store/xxxxxxxxxxxxxxxx-test
```

Then let's try to build the package again, this time using our cache. For nix-build to use our remote cache, some requirements must be satisfied:

- The builder needs to have access to the remote cache. Setup your AWS credentials
- remote path must have been signed.
- `substituters` and `trusted-public-keys` must be set, either as a flag or through `nix.conf` (see below)

Above we've made sure that we've setup AWS credentials on our machine, as well as checking that our packages in the remote cache has been properly signed.

Let's build the package again:

```sh
$ NIXOS_CACHE_PUBLIC_KEY=cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY=
$ nix-build \
  --option extra-substituters s3://fmnxl-nix-cache \
  --option trusted-public-keys $NIXOS_CACHE_PUBLIC_KEY $YOUR_PUBLIC_KEY
```

Note that I included Nix cache's public key in `trusted-public-keys`. Unfortunately there is no `extra-trusted-public-keys` flag, so if it's not included, our build will not use Nix's cache.

When we run the above, we'll see that the path will be fetched from the cache.

```
these paths will be fetched (0.00 MiB download, 0.00 MiB unpacked):
  /nix/store/whc5sw780grarhv00gpxp5iys95zxn6p-fmnxl-test
copying path '/nix/store/whc5sw780grarhv00gpxp5iys95zxn6p-fmnxl-test' from 's3://fmnxl-nix-cache'...
```

Voila!

### 5. Optimisation

#### Use nix.conf

We have been passing a number of options to commands, such as `--option substituters` and `nix sign-paths --key-file secret.key`. This becomes tedious when we're working with a lot of builds. They can be automised by adding these options to `nix.conf`.

You can find an excelent documentation on `nix.conf` [here](https://www.mankier.com/5/nix.conf). The fields we are interested in are:

**`extra-substituters`**  
You can get rid of the `--option extra-substituters` flag when running `nix-build`

**`trusted-public-keys`**  
You can get rid of the `--option trusted-public-keys` flag when running `nix-build`

**`secret-key-files`**  
By adding this config, building packages automatically signs them too, so we no longer have to `nix sign-paths` build results. Note that these have to be in the form of absolute paths (no `~`)!


Here is my `~/.config/nix/nix.conf`:

```
extra-substituters = s3://fmnxl-nix-cache?region=eu-west-1
trusted-public-keys = fmnxl-nix-cache:XxXxxxXXXXxxXXxXXXXXxxxXXXX=
secret-key-files = /home/fmnxl/fmnxl-nix-cache-keys/secret.key
```

If you're using NixOS, you can also add these in your configuration.nix instead.

```nix
{
  ...
  nix.binaryCachePublicKeys = [ "fmnxl-nix-cache:XxXxxxXXXXxxXXxXXXXXxxxXXXX=" ];
  nix.binaryCaches = [ "s3://fmnxl-nix-cache?region=eu-west-1" ];
}
```

#### Pin your Nixpkgs

When working across machines (for example, local and CI), it is possible that different versions of Nixpkgs is used. In that case, we won't be able to use the cache effectively, since different version of dependencies will result in different output hashes.

See [here](https://nixos.wiki/wiki/FAQ/Pinning_Nixpkgs) for how to pin Nixpkgs.

Basically instead of importing `<nixpkgs>`, you would be importing a `nixpkgs.nix` somewhere in your codebase.

#### Pass `--no-out-link` prevents cache invalidation.

Derivations that depend on `src` might get rebuilt when a file changes in that path. Make sure you're not making unintended changes to the source directory because a `result` symlink was created when running `nix-build`.

```
$ nix-build --no-out-link ...
```

### Troubleshooting

#### "does not have a valid signature for path"

When a path is uploaded without signing it, subsequent `nix copy` **will not** update its signature. I faced the `s3://fmnxl-nix-cache does not have a valid signature for path ...` warning uploaded because I uploaded the path without signing it.

You can check the signature of the path on the cache by passing a `--store` flag to `nix path-info`:

```
nix path-info --store s3://fmnxl-nix-cache --sigs /nix/store/xxxxx-
```

If you see that the path hasn't been properly signed, you can run `nix sign-paths` with `--store` flag to sign the remote copy of the path.

```
nix sign-paths --recursive --store s3://my-nix-cache /nix/store/xxxxxxxxxxxxxxxx-test
```

#### Access Denied

Sometimes, even after you've set your AWS credentials, the builder still complains that it can't reach `s3://fmnxl-nix-cache/nix-cache-info`. I faced this problem in CI. I'm not sure how the credentials are read by the builder, but it seems that the builder is run under a different _user_. To fix this issue:

```
$ sudo systemctl set-environment AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
$ sudo systemctl set-environment AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"
$ sudo systemctl restart nix-daemon
```

[Source](https://github.com/NixOS/nix/issues/2161#issuecomment-597845435)

## Conclusion

We were able to use a (private) cache on S3 successfully. Personally I think it isn't so difficult, if there was a good step-by-step guide. Using cachix is definitely much simpler, avoiding the need to deal with keys and editting nix.conf. But with cachix only free for public caches, I think there is a case for a tool to simplify the process of adding a private binary cache.

Would you be interested in such a tool? Comment below!
