+++
title = 'Mastering Development Workflows with Nix: Part 2 - Flakes & DevShells'
date = 2023-11-12T13:20:58-05:00
#draft = true
+++

# Introduction to devShell and Flakes

As we navigate the depths of the Nix toolkit, our focus turns to a powerful duo—devShell and Flakes. In the realm of Nix, these components emerge as robust solutions to the challenges presented by inconsistent development environments. You do not need NixOS to use nix, for instructions on installing the nix package manager on your machine look [here](https://nixos.org/download.html)

## Flakes

Flakes are just a nix file that describes your desired result, You can think of it kind of like a docker-compose file, it describebes what the result should look like not the steps to achieve it. It could be a nixos system configuration. It could be a development environment or a build process. Flakes in Nix empower developers to accomplish several critical tasks with ease. They allow you to define and manage dependencies for projects, ensuring version consistency and reproducibility across different environments. With Flakes, you can encapsulate complex configurations, including multiple packages and settings, into a single, comprehensible entity. This modular structure enables effortless sharing and reuse of configurations between projects, fostering a streamlined and efficient development process. Additionally, Flakes facilitate the creation of development environments tailored to specific projects, offering an isolated and controlled space for testing and experimentation. Overall, Flakes provide a robust foundation for orchestrating complex Nix workflows, enabling developers to architect, version, and share their projects with unparalleled precision and efficiency

## devShell: A Tailored Workspace
A devShell provides us an isolated shell environment for our project, it contains only the packages you want for the project. When used within a nix flake we get a lock file pinning each package to a sha, ensuring that we always get the same resulting environtment and build tools. DevShell provides more than just isolation—it tailors a project-specific environment effortlessly. Say goodbye to dependency headaches; with devShell, you step into a controlled space designed explicitly for your project. No more surprises during development or debugging, just a clean, isolated environment ready for action.

## devShell + Flakes
Adding our devShell within our flake gives up access to the many benefits flakes and the Nix package manager provides, such as reproducibility and and the ability to version control our environment. As we will go over in part 2 we can also use the flake to build our project, this means we can encompass not only a project specific development environtment within the flake but also the build instructions.

The integration of devShell with Flakes is all about precision in Nix workflows. In a Flakes context, devShell moves beyond basic isolation, offering meticulous organization and versioning for project environments. This streamlined alliance ensures controlled, reproducible setups, a stark departure from the uncertainties of traditional dependency management. The bottom line: Flakes and devShell together bring clarity, control, and efficiency to Nix development, making it a practical powerhouse in contrast to traditional approach

## Creating a Simple devShell within a Flake:

So to get a better idea for how this works in practice, lets construct an example(You will need the nix package manager installed).

### Step 1: Project Setup:
Navigate to your project directory.

```bash
cd path/to/your/project
```

### Step 2: Writing the flake.nix File:
Create a file named flake.nix in your project directory.
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
            ];
          };
        };
      }
    );
}
```
Lets briefly cover the components,

1. **Nixpkgs Input:**
   The `inputs.nixpkgs.url` specifies the Nixpkgs Flake input, referencing the GitHub repository and setting the reference to `nixos-unstable`.

2. **Outputs Definition:**
   The `outputs` function defines the Flake's outputs, utilizing `nixpkgs` as an argument.

3. **withSystem Function:**
   The `withSystem` function takes a function `f` and applies it to a list of systems, including "x86_64-linux," "x86_64-darwin," "aarch64-linux," and "aarch64-darwin."

4. **Fold and recursiveUpdate:**
   It employs `fold` and `recursiveUpdate` to combine results, facilitating usage on multiple system architectures, such as Apple Silicon.

5. **System Representation:**
   Within the `withSystem` function, `system` represents the individual system processed in the loop.

6. **Legacy Packages Extraction:**
   The `pkgs = nixpkgs.legacyPackages.${system}` extracts legacy packages for the specified system.

7. **devShell Definition:**
   `devShells.${system}` defines a section for the `devShell` of the specified system, potentially encompassing multiple environments. In this instance, only a default environment is provided.

8. **Default devShell with Go:**
   The `default = pkgs.mkShell { buildInputs = with pkgs; [ go ]; }` creates a `devShell` for the specified system with Go as a build input.

### Step 3: Activating the devShell:
Run the following command to activate the devShell environment:
```bash
nix develop
```
This will give us access to the packages listed in the devShell buid inputs(in this case just go). You can find additional packages [here](https://search.nixos.org/packages) by simply adding the name in the buildInputs(these are space seperated values).

### Step 4: Go Project Setup:
Let's create a basic main.go file, and remember you do not need go installed on your system, it is provided as a part of our new devShell environment:
```go
package main

import (
	"fmt"
	"net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, World!")
}

func main() {
	http.HandleFunc("/", helloHandler)

	fmt.Println("Server listening on :8000")
	http.ListenAndServe(":8000", nil)
}

```

### Step 5: Running the Go Project:
Within the devShell, you can compile and run your Go project:
```bash
go run main.go
```
Visit https://localhost:8000 to observe the functioning of your project.

### Step 6: Exiting the devShell:
To exit the devShell, simply type
```bash
exit
```

This exploration into Nix flakes and their synergy with development tools like devShell has unveiled a robust and structured approach to software development environments.

In the upcoming segment, we will delve into the practical aspects of using Nix to build your project. This will encompass leveraging Nix's capabilities to compile and construct your application, further solidifying the role of Nix as a comprehensive and indispensable tool in the developer's toolkit. As we embark on this journey of hands-on application, anticipate a seamless integration of Nix into your workflow, transforming the once arduous task of building projects into an efficient and dependable process. Stay tuned for an in-depth exploration of Nix's building prowess and witness the transformative impact it can bring to your development pipeline.
