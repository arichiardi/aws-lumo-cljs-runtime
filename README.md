This repo contains an implementation of a [custom AWS Lambda runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html)
that enables executions of [ClojureScript](http://clojurescript.org) code in AWS Lambda without any pre-compilation.

It relies on the awesome [Lumo](https://github.com/anmonteiro/lumo) project, and
was inspired by [this episode of The Repl podcast](https://www.therepl.net/episodes/14/).

It's based on the [Tutorial from Amazon](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-walkthrough.html)
as well as the [Node 10.x/11.x implementations from LambCI](https://github.com/lambci/node-custom-lambda).

It's still very alpha, but this document contains a step-by-step guide for getting things started.

### Compiling Lumo

In order to interpret ClojureScript in an AWS Lambda context, you need a static build of Lumo with all libraries included.

If are curious about the nitty gritty details, check the following blog post:

 * [Compile the Lumo ClojureScript interpreter for AWS Lambda](https://andrearichiardi.com/blog/posts/aws-lambda-runtime-lumo.html)

First things first, pull down [the docker image](https://cloud.docker.com/repository/docker/arichiardi/lumo-musl-ami):

```shell
docker pull arichiardi/lumo-musl-ami
```

Second, clone `lumo`:

```shell
git clone git@github.com:anmonteiro/lumo   # anywhere on your filesystem
```

Finally, build using the image:

```shell
docker run -v /path/to/lumo:/lumo -v /home/user/.m2:/root/.m2 -v /home/user/.boot/cache:/.boot/cache --rm arichiardi/lumo-musl-ami
```

The `/root/.m2` and `/.boot/cache` mappings are optional but recommended for
avoiding downloading dependencies multiple times.

The (long) process will compile the lumo static binary under `build`.

### Create the runtime archive

```shell
zip -j runtime.zip bootstrap /path/to/build/lumo runtime.cljs
```

The flag `-j` just ignores paths and puts everything in the archive root.

### Publish layer

A layer can be used by a lambda to pull in additional code. In this context, the layer contains the actual runtime:

```shell
aws lambda publish-layer-version --layer-name lumo-runtime --zip-file fileb://runtime.zip
```

You'll get an `arn` with a layer version back, which you'll need when configurating the lambda.

#### Using the Makefile

For convenience, the above operations can also be executed with `make`:

```shell
make clean (will delete the runtime.zip if necessary)
LUMO_BIN_PATH=/path/to/build/lumo make
make publish
```

### Try the demo lambda

#### Create function archive

```shell
zip -r function.zip my_package
```

The `my_package` dir in this repo contains a simple handler, but you can provide your own.

#### Upload and invoke

For the `--role` parameter, you must supply a role that can execute lambdas.
See https://docs.aws.amazon.com/lambda/latest/dg/runtimes-walkthrough.html#runtimes-walkthrough-prereqs

The `--handler` parameter must correspond to the directory structure of the ClojureScript code that you provide:

```shell
aws lambda create-function --function-name test-lumo --zip-file fileb://function.zip --handler my-package.my-ns/my-handler --runtime provided --role arn:aws:iam::xxx:role/lambda-role
```

Use the layer `arn` that you received when publishing the layer, including the layer version, to configure the lambda:

```shell
aws lambda update-function-configuration --function-name test-lumo --layers arn:aws:lambda:eu-west-1:xxx:layer:lumo-runtime:1
```

#### Invoke the lambda

```shell
aws lambda invoke --function-name test-lumo --payload '{"foo":42}' response.txt
```

You should receive something like this in `response.txt`:

```json
{
  "hello": "Hello from my-handler!",
  "input": {
    "event": {
      "foo": 42
    },
    "context": {
      "aws-request-id": "b64259ce-03e0-11e9-8db3-1bbff8d08d21",
      "lambda-runtime-invoked-function-arn": "arn:aws:lambda:eu-west-1:xxx:function:test-lumo"
    }
  }
}
```
