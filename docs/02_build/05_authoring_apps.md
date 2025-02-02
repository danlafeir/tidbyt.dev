# Authoring Apps

This guide provides best practices on how to build apps to integrate well with
the Tidbyt ecosystem.

## Architecture
Pixlet is heavily influenced by the way Tidbyt devices work. The Tidbyt displays tiny 64x32 animations and images. It keeps a local cache of the apps that are installed on it.

Each Tidbyt regularly sends heartbeats to the Tidbyt cloud, announcing what it has cached. The Tidbyt cloud decides which app needs to be rendered next, and executes the appropriate Starlark script. It encodes the result as a [WebP](https://developers.google.com/speed/webp) image, and sends it back to the device.

To mimic how we host apps internally, `pixlet render` executes the Starlark script and `pixlet push` pushes the resulting WebP to your Tidbyt.

## Config
When running an app, Pixlet passes a `config` object to the app's `main()`:

```starlark
def main(config):
    who = config.get("who")
    print("Hello, %s" % who)
```

The `config` object contains values that are useful for your app. You can set the actual values by:

1. Passing URL query parameters when using `pixlet serve`.
2. Setting command-line arguments via `pixlet render`.

When apps are published to the [Tidbyt Community repo][3], users can install and configure them with the Tidbyt smartphone app. [Define a schema for your app][4] to enable this.

Your app should always be able to render, even if a config value isn't provided. Provide defaults for every config field, or check if the value is `None`. This will ensure the app behaves as expected even if config was not provided.

For example, the following ensures there will always be a value for `who`:

```starlark
DEFAULT_WHO = "world"

def main(config):
    who = config.get("who", DEFAULT_WHO)
    print("Hello, %s" % who)
```

The `config` object also has helpers to convert config values into specific types:

```starlark
config.str("foo") # returns a string, or None if not found
config.bool("foo") # returns a boolean (True or False), or None if not found
```

## Caching

Many apps retrieve data from external services and APIs. It's
important both to be mindful of the request rates hitting these
services, and of the possibility of bad internet weather causing some
requests to fail. Caching plays an important role here.

For the majority of apps, it's sufficient to rely on the built-in
cache functionality of the HTTP module. Simply pass in `ttl_seconds`
when making a requests, and the HTTP response will be cached
accordingly. This allows subsequent requests (until the TTL has
expired) to be served quickly, without hammering the upstream
API.

## Secrets

Many apps need secret values like API keys. When publishing your app to the [Tidbyt community repo][3], encrypt sensitive values so that only the Tidbyt cloud servers can decrypt them.

To encrypt values, use the `pixlet encrypt` command. For example:

```shell
# replace "googletraffic" with the folder name of your app in the community repo
$ pixlet encrypt googletraffic top_secret_google_api_key_123456
"AV6+...."  # encrypted value
```

Use the `secret.decrypt()` function in your app to decrypt this value:

```starlark
load("secret.star", "secret")

def main(config):
    api_key = secret.decrypt("AV6+...") or config.get("dev_api_key")
```

When you run `pixlet` locally, `secret.decrypt` will always return `None`. When your app runs in the Tidbyt cloud, `secret.decrypt` will return the string that you passed to `pixlet encrypt`.


## Fail
The [`fail()`][1] function will immediately end the execution of your app and return an error. It should be used incredibly sparingly, and only in cases that are _permanent_ failures. 

For example, if your app receives an error from an external API, try these options before `fail()`:

1. Return a cached response.
2. Display a useful message or fallback data.
3. [`print()`][2] an error message.
3. Handle the error in a way that makes sense for your app.

[1]: https://github.com/bazelbuild/starlark/blob/master/spec.md#fail
[2]: https://github.com/bazelbuild/starlark/blob/master/spec.md#print
[3]: https://github.com/tidbyt/community
[4]: ../06_reference/schema.md

## Performance profiling

Some apps may take a long time to render, particularly if they produce a long and complex animation. You can use `pixlet profile` to identify how to optimize the app's performance. Most apps will not need this kind of optimization.

```shell
$ pixlet profile path_to_your_app.star
```

When you profile your app, it will print a list of the functions which consume the most CPU time. Improving these will have the biggest impact on overall run time.