<picture>
    <source srcset="https://raw.githubusercontent.com/leptos-rs/leptos/main/docs/logos/Leptos_logo_Solid_White.svg" media="(prefers-color-scheme: dark)">
    <img src="https://raw.githubusercontent.com/leptos-rs/leptos/main/docs/logos/Leptos_logo_RGB.svg" alt="Leptos Logo">
</picture>

# Leptos Starter Template for AWS Lambda

This is a template for the [Leptos](https://github.com/leptos-rs/leptos) web framework managed with [cargo-leptos](https://github.com/akesson/cargo-leptos) and integrated with [Axum](https://github.com/tokio-rs/axum) for deployment on [AWS Lambda](https://aws.amazon.com/lambda/).

## Creating your template repo

If you don't have `cargo-leptos` installed you can install it with

```bash
cargo install cargo-leptos
```

Then either click the "Use this template" button in the Github UI or run

```bash
cargo generate --git https://github.com/leptos-rs/start-aws.git
```

to generate a new project template.

```bash
cd {{project-name}}
```

to go to your newly created project.
Feel free to explore the project structure, but the best place to start with your application code is in `src/app.rs`.
Additionally, Cargo.toml may need updating as new versions of the dependencies are released, especially if things are not working after a `cargo update`.

## Running your project

```bash
cargo leptos watch
```

The `--release` flag enables the [axum-aws-lambda](https://github.com/lazear/axum-aws-lambda) layer,
which only works within the AWS Lambda environment,
so it should be avoided locally.

## Installing Additional Tools

By default, `cargo-leptos` uses `nightly` Rust, `cargo-generate`, and `sass`. If you run into any trouble, you may need to install one or more of these tools.

1. `rustup toolchain install nightly --allow-downgrade` - make sure you have Rust nightly
2. `rustup target add wasm32-unknown-unknown` - add the ability to compile Rust to WebAssembly
3. `cargo install cargo-generate` - install `cargo-generate` binary (should be installed automatically in future)
4. `npm install -g sass` - install `dart-sass` (should be optional in future

## Compiling for Release

```bash
cargo leptos build --release
```

Will generate your server binary in target/server/release and your site package in target/site

## Testing Your Project

```bash
cargo leptos end-to-end
```

```bash
cargo leptos end-to-end --release
```

Cargo-leptos uses Playwright as the end-to-end test tool.
Tests are located in end2end/tests directory.

## Deploying Your Project

To build and deploy your project to AWS, you'll need [cargo-lambda](https://www.cargo-lambda.info/).
They provide [installation instructions](https://www.cargo-lambda.info/guide/installation.html) on their site.

Let's start by building the project with `cargo-leptos`:

```bash
cargo leptos build --release
```

We won't use the server binary that it builds, since
the Lambda function requires a particular architecture that
`cargo-lambda` will handle for us. If you'd rather not build the server twice,
you'll have to manage the WASM build and optimization yourself.

Next, let's build the production server binary:

```bash
LEPTOS_OUTPUT_NAME={{project-name}} cargo lambda build --no-default-features --features=ssr --release
```

This should produce a binary at `target/lambda/{{project-name}}/bootstrap`.
`Cargo.toml` exposes all the required environment variables to `cargo-lambda`
so that the server can run in production.

Finally, we can deploy the project to AWS:

```bash
cargo lambda deploy --include target/site --enable-function-url
```

After a few seconds, `cargo-lambda` should print out the URL of your deployed function!

This template provides a GitHub action that handles all these steps automatically.
To use it, you'll need to provide your AWS credentials as [repository secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions).
You'll also need to provide the project name as a variable.

## Notes

### Credentials

You'll need AWS credentials with some permissions for IAM and Lambda operations.
`cargo-lambda` provides the [minimum requirements here](https://www.cargo-lambda.info/commands/deploy.html#user-profile).

Setting up permissions can be a bit onerous if this is your first time working with AWS.
For a quick and dirty setup, you can:

1. Create a new user in the IAM service (Access Management > Users)
2. Click "Attach policies directly" on the "Set permissions" page
3. Add the "AWSLambda_FullAccess" and "IAMFullAccess" policies, and complete the user creation
4. Create an access key for the user, with "Use case" "Application running on an AWS compute service" (don't worry about the warning - just confirm)
5. Save your credentials in an `~/.aws/credentials` file (or the appropriate location for your system, for `cargo-lambda` deployments),
or store the credentials in a password manager for later setting up CI/CD deploys; your AWS access key and secret key will look similar to the (randomly generated) example, below.

```
[default]
aws_access_key_id = AKIAQYLPMN5HCTNK35FD
aws_secret_access_key = rbWHpaI/lJnXdLteWHNnTVZpQztMB2+pdbb+KVgr
```

To use Github Actions to deploy to AWS Lambda, add the following variables and secrets to Github by going to your repo's "Settings" > "Secrets and Variables" > "Actions"
and enter in the following:

Variables:

- LEPTOS_OUTPUT_NAME - the name of your project, as in the `Cargo.toml` file
- AWS_DEFAULT_REGION - the AWS region you want your Lambda deployed to, eg. `us-west-2`

Secrets:

- AWS_ACCESS_KEY_ID - from AWS IAM credentials
- AWS_SECRET_ACCESS_KEY - the secret key associated with the Access Key, from the AWS IAM step (above)

### State

Since AWS Lambda is a serverless platform, you'll need to be more careful about how
you manage long-lived state. Writing to disk or using an axum state extractor will
not work reliably across requests. Instead, you'll need a database or other
microservices that you can query from the Lambda function.

### Optimizations

Serving static files from a lambda function is not the best approach.
Ideally, you should upload your files to a CDN and configure
your project to serve them from that location.
AWS has an article on deploying [React with SSR](https://aws.amazon.com/blogs/compute/building-server-side-rendering-for-react-in-aws-lambda/).

It's also pretty easy to set up edge compute with Lambda@Edge,
which should improve latency for users around the world.
