+++
title = 'Mastering Development Workflows with Nix: Part 2 - Building Your Project with Nix'
date = 2023-11-14T13:17:00-05:00
#draft = true
+++

In the previous part, we walked through setting up a basic Go project using a nix flake. We setup our development environment with a devShell. Now, let's dive into the process of building and running this project using the Nix Flake.

## Building the Project

Now that we have meticulously set up our development environment using the Nix Flake, the next crucial step is to build our Go project. In this section, we'll walk through the process of constructing the project, leveraging the capabilities of Nix to ensure a reliable and reproducible build. Let's dive into the details of how the Nix Flake orchestrates the build process for our Go project.

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

The new part we added is the defaultPackage section. Lets go over what this means.

1. **`defaultPackage.${system}`:**
   Defines a section for creating a package with the name `my-go-project` for the specified system.

2. **`name = "my-go-project";`:**
   Sets the name of the resulting package to "my-go-project."

3. **`src = ./.;`:**
   Specifies the source directory for the package. In this case, it is set to the current directory (`./`), indicating that the source code for the Go project is in the same directory as the `flake.nix` file.

4. **`buildInputs = [ pkgs.go ];`:**
   Lists the dependencies required to build the package. Here, it includes the `go` package from Nixpkgs, ensuring that the Go compiler is available during the build process.

5. **`buildPhase = '' ... '';`:**
   Contains the shell script that defines the actual build process. It is a sequence of shell commands executed in the build environment.

6. **`export GOCACHE=$TMPDIR/go-cache`:**
   Sets the `GOCACHE` environment variable to a directory inside the temporary directory (`$TMPDIR`). `GOCACHE` determines the location of the Go build cache.

7. **`mkdir -p $TMPDIR/go-cache`:**
   Creates the `go-cache` directory if it doesn't exist. This is the location where the Go build cache will be stored.

8. **`go build -o $out/my-go-project ./main.go`:**
   The actual Go build command. It compiles the `main.go` file in the current directory (`./`) and produces an executable named `my-go-project`. The `-o $out/my-go-project` specifies the output directory in the Nix store where the resulting binary will be placed.

In summary, this section defines how to create a Nix derivation for the Go project, specifying the build process, dependencies, and resulting executable. The use of environment variables and careful directory management ensures a reproducible and isolated build process.

## Examining the Build Output

After a successful build, it's essential to understand the conventions of Nix. The result symlink, which you'll find in the project directory, is a standard mechanism in Nix for conveniently accessing the outputs of the build. This symlink points directly to the directory containing the build results, providing a straightforward way to locate and utilize the compiled artifacts of your Go project. Let's take a moment to explore how this standard convention simplifies the examination and utilization of the build output.
```bash
ls -l result
```
Running the Project

Now, you can run the resulting executable as normal:
```bash
./result/my-go-project
```
## Reproducibility and Consistency

One of the distinctive strengths of Nix Flakes lies in their ability to capture the entire dependency tree of a project. As we introduced the Flake for our Go project, every library, tool, and configuration is meticulously documented and defined within the Flake. This comprehensive representation ensures that every aspect of the development environment, from dependencies to build processes, is encapsulated.

By encapsulating the dependency tree, Nix Flakes provide a powerful mechanism for ensuring consistency and reproducibility across diverse development environments. This means that regardless of the system or machine where the project is built or executed, the Flake guarantees that the exact same set of dependencies and configurations is utilized. This level of precision is crucial for collaborative projects, as it mitigates the notorious "It works on my machine" scenario, offering a reliable and consistent foundation for developers to collaborate and build upon.

In essence, the Flake acts as a blueprint, capturing not only the explicit dependencies but also the relationships and interactions among them. This holistic approach to defining the development environment is instrumental in minimizing discrepancies between various systems, ultimately contributing to a smoother and more reliable development experience.

## Conclusion

In conclusion, our journey through Part 2 has shed light on the formidable capabilities of Nix Flakes, particularly in the realm of building projects. The meticulous orchestration of dependencies, configurations, and the build process within the Flake guarantees a reproducible and consistent development environment specifically tailored for constructing our Go project.

Nix Flakes, in this context, emerge as a powerful tool for streamlining the sharing and reproduction of development environments across different systems during the build phase. Whether collaborating with team members, deploying applications to diverse servers, or transitioning between personal machines, the Flake acts as a reliable blueprint, ensuring that the build process is identical under various conditions.

This focus on reproducibility and consistency is especially vital during the build phase, where discrepancies can lead to unforeseen issues. By leveraging Nix Flakes, developers gain a robust framework for managing dependencies and architecting a build workflow that remains impervious to the inconsistencies that often plague software projects.

As we delve deeper into the capabilities of Nix Flakes, let's appreciate the transformative impact they bring to the build process. In the upcoming parts of this series, we will continue our exploration, extending our understanding to additional aspects such as Continuous Integration (CI) and the seamless integration of Nix Flakes in Docker containers. Stay tuned for a more comprehensive journey into the versatile world of Nix Flakes.
