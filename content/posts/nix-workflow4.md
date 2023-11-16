+++
title = 'Mastering Development Workflows with Nix: Part 4 - Automating Builds with Nix and Continuous Integration'
date = 2023-11-15T16:54:55-05:00
#draft = true
+++

In the previous posts, we've set up a reproducible development environment for our Go project using Nix. Now, let's take it a step further by integrating Continuous Integration (CI) into the mix. CI ensures that our project is automatically built, tested, and validated whenever changes are pushed to the repository.

## Embracing Nix in Continuous Integration: Ensuring Reproducibility and More

In the realm of modern software development, adopting robust CI practices is paramount. When coupled with Nix, the benefits extend beyond automated testing and builds. Let's delve into why incorporating Nix into your CI workflow is a game-changer.

### Environment Reproducibility:

One of the standout features of Nix is its ability to create reproducible development environments. In a typical CI process, discrepancies in the development environment can lead to inconsistencies in builds and tests. With Nix, you define your project dependencies declaratively, ensuring that the exact same environment is replicated across different stages of your CI pipeline. This reproducibility minimizes the notorious "It works on my machine" problem, providing a consistent environment for both development and CI.

### Dependency Management Made Easy:

Nix simplifies the often challenging task of managing project dependencies. By encapsulating dependencies in a Nix Flake or expression, you create a self-contained environment that avoids version conflicts and facilitates easy sharing of development setups. This approach not only streamlines the CI configuration but also makes your project more approachable for collaborators and contributors.

### Deterministic Builds:

In CI, the goal is to have predictable and deterministic builds. Nix achieves this by creating isolated build environments with precisely defined dependencies. This deterministic nature ensures that your builds are not affected by external changes, providing confidence that your CI pipeline produces consistent results.

### Easy Integration with Multiple Systems:

Nix supports multiple systems and architectures, making it effortless to extend your CI workflow across various platforms. Whether you're building for Linux, macOS, or different CPU architectures, Nix abstracts away the complexities, enabling seamless integration with diverse environments.

### Version Pinned Dependencies:

Nix allows you to specify exact versions for your project's dependencies, reducing the risk of unexpected behavior caused by unforeseen updates. This version-pinning strategy contributes to the stability of your CI builds, as changes in external dependencies won't catch you off guard.

## Choosing a CI Platform

Before diving into the specifics, choose a CI platform that supports Nix. Some popular options include GitHub Actions, GitLab CI, and Travis CI. In this example, we'll use GitHub Actions.

## Setting Up GitHub Actions
### Create a .github/workflows directory in your project.
```bash
mkdir -p .github/workflows
```
### Create a build.yml file inside .github/workflows.
```yaml
name: Nix Actions Example

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Nix
        uses: cachix/install-nix-action@v12
```
This is the basis of our workflow. It will allow us to use `nix` commands for future steps.

## Adding golint to Your Go Project

In the world of Go development, maintaining a clean and consistent codebase is crucial. One tool that helps enforce coding standards and style conventions is golint. In this section, we'll walk through the process of integrating golint into your Go project's Continuous Integration (CI) workflow.

### Why golint?

golint is a linter for Go source code that checks for common style mistakes and deviations from the Go coding conventions. By incorporating golint into your CI pipeline, you can catch potential issues early in the development process, ensuring a higher level of code quality and adherence to best practices.
Updating Your CI Workflow

First lets update the devShell in our flake to include golint.

```nix
{
  description = "A simple go flake";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs?ref=nixos-unstable";

  outputs =
    { nixpkgs, ... }:
    let
      withSystem =
        f:
        nixpkgs.lib.fold nixpkgs.lib.recursiveUpdate { } (
          map f [
            "x86_64-linux"
            "x86_64-darwin"
            "aarch64-linux"
            "aarch64-darwin"
          ]
        );
    in
    withSystem (
      system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
      in
      {
        devShells.${system} = {
          default = pkgs.mkShell {
            buildInputs = with pkgs; [
              go
              golint
            ];
          };
        };
        defaultPackage.${system} = pkgs.stdenv.mkDerivation {
          name = "my-go-project";
          src = ./.;
          buildInputs = [ pkgs.go ];
          buildPhase = ''
            export GOCACHE=$TMPDIR/go-cache
            mkdir -p $TMPDIR/go-cache
            go build -o $out/my-go-project ./main.go
          '';
        };
      }
    );
}
```
Now if we enter our devShell we can run golint manually as usual. But we want to run it to run on github. to do that lets update our build.yml

```yaml
name: Nix Actions Example

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Nix
        uses: cachix/install-nix-action@v12

      - name: Lint with golint
        run: nix develop --command golint .
```
`nix develop --command` lets you run a command using the devShell environment without actually entering the environment, in this case we want to run golint.

## Elevating Local Development Confidence with Nix and Scripts

In the pursuit of a reproducible development environment, we've embraced Nix to ensure that our GitHub Actions workflow and local development environments mirror each other. This alignment guarantees that if our code builds and tests successfully on our machine, it will do the same in our CI pipeline.

Yet, when working locally, it's not always apparent which commands to run to simulate the CI process accurately. To bridge this gap and elevate our confidence in local development, we can enhance our Nix Flake by incorporating scripts.

## Scripting for Local Consistency

Scripts provide a clear and concise way to encapsulate the steps required for linting, building, and testing our Go project. By embedding these scripts within our Flake, we create a unified interface for developers. Now, instead of searching through CI workflows, one can simply run a predefined script to ensure local compatibility.

## An Example Integration

Let's take linting with golint as an example. By including a script in our Flake that encapsulates the golint command, developers can effortlessly lint their code locally with the same tool and version used in the CI workflow.

```nix
custom_lint = pkgs.writeShellApplication {
  name = "my-lint-script";
  text = ''
    #!/bin/sh
    echo "Running golint..."
    golint .
  '';
};
```
This script, when executed, echoes a message and runs golint on the current directory.

## Integrating the Custom Lint Script into the Development Environment

To integrate it into the development environment provided by the Nix Flake. Within the devShells section, we extend the list of build inputs to include our custom lint script along with the standard Go tools:

```nix
devShells.${system} = {
  default = pkgs.mkShell {
    buildInputs = [ custom_lint ] ++ (with pkgs; [
      go
      golint
    ]);
  };
};
```
The flake will now look like this
```nix
{
  description = "A simple go flake";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs?ref=nixos-unstable";

  outputs =
    { nixpkgs, ... }:
    let
      withSystem =
        f:
        nixpkgs.lib.fold nixpkgs.lib.recursiveUpdate { } (
          map f [
            "x86_64-linux"
            "x86_64-darwin"
            "aarch64-linux"
            "aarch64-darwin"
          ]
        );
    in
    withSystem (
      system:
      let
        pkgs = nixpkgs.legacyPackages.${system};

        custom_lint = pkgs.writeShellApplication {
          name = "my-lint-script";
          text = ''
            #!/bin/sh
            echo "Running golint..."
            golint .
          '';
        };
      in
      {
        devShells.${system} = {
          default = pkgs.mkShell {
            buildInputs = [ custom_lint ] ++ (with pkgs; [
              go
              golint
            ]);
          };
        };
        defaultPackage.${system} = pkgs.stdenv.mkDerivation {
          name = "a-go-example";
          src = ./.;
          buildInputs = [ pkgs.go ];
          buildPhase = ''
            export GOCACHE=$TMPDIR/go-cache
            mkdir -p $TMPDIR/go-cache
            go build -o $out/a-go-project ./main.go
          '';
        };
      }
    );
}
```

With this configuration, anyone using the Nix Flake to enter the development environment (devShell) will not only have access to Go and golint but can also just call our new script using `my-lint-script`(this would be more usefull if we used more flags or a more complex process).

We can now call this script in our github workflow much like we did above but first lets add some unit tests so we can run them too.

```go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHelloHandler(t *testing.T) {
	// Create a request to the helloHandler
	req, err := http.NewRequest("GET", "/", nil)
	if err != nil {
		t.Fatal(err)
	}

	// Create a ResponseRecorder (which implements http.ResponseWriter) to record the response
	rr := httptest.NewRecorder()

	// Call the helloHandler with the ResponseRecorder and the Request
	helloHandler(rr, req)

	// Check the status code
	if status := rr.Code; status != http.StatusOK {
		t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
	}

	// Check the response body
	expected := "Hello, World!"
	if rr.Body.String() != expected {
		t.Errorf("handler returned unexpected body: got %v want %v", rr.Body.String(), expected)
	}
}
```
we can run these manually inside our devShell with `go test`

Lets add a script like above to call this too.

```nix
        some_tests = pkgs.writeShellApplication {
          name = "my-test-script";
          text = ''
            #!/bin/sh
            echo "Running tests..."
            go test
          '';
        };
```

Now we can update our github workflow to use these new scripts.

```yaml
name: Build, Test, and Lint

on:
  push:
    branches:
      - main

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Nix
        uses: cachix/install-nix-action@v23

      - name: Lint Go code
        run: nix develop --command my-lint-script

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Nix
        uses: cachix/install-nix-action@v23

      - name: Run tests
        run: nix develop --command my-test-script
```
Lastly we want to include the build step. We have already covered nix build so we just need to add another step to our workflow.
```yaml
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Nix
        uses: cachix/install-nix-action@v23

      - name: Build
        run: nix build

      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: a-go-project
          path: result/

```
we can add this step to simply upload the resulting build as an artifact when our actions run(releases and semvar are out of scope for this guide so we opt for a simple approach).

Lastly while this works we dont want to have to call a script for each individual step. So lets write another script.in
```nix
        ci_checks = pkgs.writeShellApplication {
          name = "my-ci-checks";
          text = ''
            #!/bin/sh
            echo "Running all CI checks..."
            my-lint-script
            my-test-script
          '';
        };
```
the resulting flake should look like this
```nix
{
  description = "a simple go flake";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs?ref=nixos-unstable";

  outputs =
    { nixpkgs, ... }:
    let
      withSystem =
        f:
        nixpkgs.lib.fold nixpkgs.lib.recursiveUpdate { } (
          map f [
            "x86_64-linux"
            "x86_64-darwin"
            "aarch64-linux"
            "aarch64-darwin"
          ]
        );
    in
    withSystem (
      system:
      let
        pkgs = nixpkgs.legacyPackages.${system};

        custom_lint = pkgs.writeShellApplication {
          name = "my-lint-script";
          text = ''
            #!/bin/sh
            echo "Running golint..."
            golint .
          '';
        };

        some_tests = pkgs.writeShellApplication {
          name = "my-test-script";
          text = ''
            #!/bin/sh
            echo "Running tests..."
            go test
          '';
        };

        ci_checks = pkgs.writeShellApplication {
          name = "my-ci-checks";
          text = ''
            #!/bin/sh
            echo "Running all CI checks..."
            my-lint-script
            my-test-script
            echo "Building..."
            nix build
          '';
        };

      in
      {
        devShells.${system} = {
          default = pkgs.mkShell {
            buildInputs = [ custom_lint some_tests ci_checks ] ++ (with pkgs; [
              go
              golint
            ]);
          };
        };
        defaultPackage.${system} = pkgs.stdenv.mkDerivation {
          name = "a-go-example";
          src = ./.;
          buildInputs = [ pkgs.go ];
          buildPhase = ''
            export GOCACHE=$TMPDIR/go-cache
            mkdir -p $TMPDIR/go-cache
            go build -o $out/a-go-project ./main.go
          '';
        };
      }
    );
}
```

With our Nix Flake configured to streamline our CI workflow, local development becomes even more seamless. Now, by executing the command `my-ci-checks` within our devShell, we can efficiently run all essential CI checks. This proactive approach empowers developers to identify and resolve potential issues before pushing changes or submitting pull requests. Embracing this practice enhances code quality and ensures a smoother integration process in the CI pipeline.

## Conclusion

In this second part, we explored the intricacies of the Nix Flake, focusing on building our Go project with confidence and reliability. By leveraging Nix, we not only achieved a reproducible development environment but also ensured consistency in our CI workflow. It's worth noting that Nix Flakes extend beyond managing builds, offering a versatile and declarative approach to various aspects of software development. Stay tuned for upcoming articles where we'll delve into more advanced use cases of Nix Flakes, unlocking their full potential.

You can find code used in these examples [here](https://github.com/gwg313/blog-go-example).

