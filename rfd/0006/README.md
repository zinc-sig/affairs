---
authors: David Mak, Kristopher Lam
state: prediscussion
discussion:
labels: process
---

# Grading Configuration 

This RFD intends to establish the intended format for the domain-specific language (DSL) for grading configurations, which will be used by teaching staff to specify the process of which student submissions will be graded.

## Background

In ZINC 1.0, the grading configuration is specified by a YAML Ain't Markup Language (YAML) file. The file consists of a `_settings` block, which specifies pipeline-wide settings, followed by one or more stage definitions which are executed sequentially. More information regarding the structure and grammar of the 1.0 grading configuration can be found [here](https://docs.zinc.ust.dev/docs/sphinx/user/model/Config.html).

An example of the grading configuration is shown below (taken from [here](https://docs.zinc.ust.dev/docs/sphinx/user/writing-config-yaml-cpp.html)).

```yaml
_settings:
  lang: cpp/g++:8
  use_template: FILENAMES
  template:
    - do_not_include.txt
  use_skeleton: true
  use_provided: true
  stage_wait_duration_secs: 10
  cpus: 1.0
  mem_gb: 2.0
  enable_features:
    network: false

fileStructureValidation: {}

diffWithSkeleton: {}

compile:
	# ...

stdioTest:
	# ...

score:
	# ...
```

However, this design suffers from the following issues:

1. Insufficient configuration granularity

Some fields in `_settings`, such as `stage_wait_duration_secs`, `cpus`, `mem_gb`, and `enable_features`, can benefit from having per-stage toggles, since the requirements for each stage will differ (e.g. `diffWithSkeleton` should never take more than 5s to execute, while Machine Learning-related stages may take upwards of minutes to complete).

2. No capability to reuse previously defined stages

Some courses may want to reuse stages that have previously been defined within a course. However, the current grading configuration will require teaching staff to 1. ask for previous teaching staff to retrieve old grading configurations, and 2. extract the relevant stage from the old configuration and adapt it for this configuration.

3. Stage design is not extensible

Current stage designs are all defined as individual classes, with little code sharing between each other. This causes increased maintenance as developers will need to maintain a large set of stages with different stage definitions, while adding a stage will require modifying the codebase.

4. No capability to introduce batch-wide stages

Some functionality, e.g. JPlag plagiarism checking, cannot be specified in the grading configuration as there is no support for stages to perform cross-submission checking.

## Proposal

This proposal will be separated into three sections: One describing the format specification for stage templates, one describing the format specifications for the grading configuration, and one describing an intermediate representation used internally by the Grader.

Unless otherwise specified, all configurations below are defined using [HashiCorp Configuration Language (HCL)](https://github.com/hashicorp/hcl).

## Stage Template

A stage template is a partial definition of a stage. It is analogous to the `PipelineStage`s defined in Grader 1.0.

### Stage Template Configuration

A template shall be defined in `env.hcl` at the top-level of a public Git repository. A template is defined by a `template` block, followed by a block label with the link of the Git repository housing this file.

```terraform
// github.com/zinc-sig/contrib/gcc-compile:/env.hcl
template "github.com/zinc-sig/contrib/gcc-compile" {
	// ...
}
```

The `template` block should contain a `setup` attribute, which specifies zero or more commands that will be executed during the generation of the image template.

```terraform
template "github.com/zinc-sig/contrib/gcc-compile" {
	setup = <<EOT
		nix install gcc
	EOT
}
```

The above `setup` attribute may generate the following equivalent `Dockerfile`:

```Dockerfile
FROM nixos/nix:latest
RUN nix-channel --update
RUN nix install gcc
```

The `template` block may contain an `unstable` attribute, which specifies whether to use NixOS's unstable branch.

```terraform
template "github.com/zinc-sig/contrib/gcc-compile" {
	unstable = true
}
```

The above `unstable` attribute may generate the following equivalent `Dockerfile`:

```Dockerfile
FROM nixos/nix:latest
RUN nix-channel --add https://nixos.org/channels/unstable nixos
RUN nix-channel --update
```

The `template` block may contain an `image` attribute, which specifies the hash of the built image that was constructed as specified by other attributes of this block. This attribute shall be used for internal caching use by the orchestrator, and shall not be used by the end-user.

### Stage Template Evaluation Implementation

The Git repository may also contain an *evaluation implementation*. An evaluation implementation specifies the evaluation logic when the Grader is executing a stage template with a student submission as its input.

The evaluation implementation shall consist of one or more Go files, effectively acting as a single Go package. The Go package shall contain exactly one implementation of the following Go function, and zero or more helper implementations.

```go
func Grade(opts: map[string]interface, context; ctx.Context, scenarios: *Scenarios) *Result, Error {
	// ...
}
```

The `Grade` function shall accept the following arguments:

- `opts`: Options that modify the execution behavior of this stage.
- `context`: Context associated with the entire pipeline, including previous stage results, helper file information, etc.
- `scenario`: Abstract rules that can be applied to the execution output to map and/or modify the stage output(s)

```go
import "os"

func Grade(opts: map[string]interface, context; ctx.Context, scenarios: *Scenarios) *Result, Error {
	// runtime check keys

	for _, s := range scenarios {
		stdout, stderr := os.Exec(...)
		// rvars = Reserved Variables used in scenario def
		used_rvars := /* Unmarhsal scenarios and read reserved keywords, repack data structures */
		s.Evaluate(used_rvars)
	}

	// Result = {STDIN, STDOUT, STDERR, convertToInternalRepr(...)}
}
```

## Grading Configuration

The grading configuration will be defined by an HashiCorp Configuration Language (HCL) file. 

#### Stage Definition

#### `use` Block

Stage definitions must

#### `option` Block

Stage definitions shall contain an `option` block that modifies the default behavior for a stage.

### Intermediate Representation (IR)

The goal of the Grading Configuration IR is to resolve all constraints defined in the grading configuration, so that the individual stages can use the grading configuration without the need to consult its parent context (i.e. other stage information) to perform its grading task.

## Example

```terraform
pipeline "pa1" {
    stage "compile" {
      use "contrib/gcc-compile" {
        options {
          flags = ["-W"]
        }
		
        on "failure" {
          
        }
      }
    }

    stage "test" {
      allow_replay = true
      use "contrib/stdio-test" {
        options {
          ignore_whitespace = true
          timeout = "5s"
          input "" {
            
          }
        }
      }
    }
    
    stage "plagerism-check" {
      scope = "all"
      use "contrib/jplag" {
        options {
          
        }
      }
    }
}
```

v2 (2025-04-05):

```terraform
// github.com/zinc-sig/contrib/gcc-compile:/env.hcl
template "github.com/zinc-sig/contrib/gcc-compile" {
	// implicit: nixos/nix:latest
	image = "<image_hash>"
	// or
	setup = <<EOT
		nix install gcc
	EOT
}

// github.com/zinc-sig/contrib/stdio-test:/env.hcl
template "github.com/zinc-sig/contrib/stdio-test" {
	setup = <<EOT
		nix install libstdc++
	EOT
}

// github.com/zinc-sig/contrib/stdio-test:/main.go
import "os"

func Grade(opts: map[string]interface, context: ctx.Context, scenarios: *Scenarios) *Result {
	// runtime check keys

	// Fork repo to implement preExec actions

	for _, s := range scenarios {
		stdout, stderr := os.Exec(...)
		// rvars = Reserved Variables used in scenario def
		used_rvars := /* Unmarhsal scenarios and read reserved keywords, repack data structures */
		s.Evaluate(used_rvars)
	}

	// Fork repo to implement postExec actions

	// Result = {STDIN, STDOUT, STDERR, convertToInternalRepr(...)}
}

// grading script
pipeline "pa1" {
	defaults {
		cpus = 1.0
		mem_gb = 2.0
		/* stage_wait_duration_secs = nil */
		capabilities {
			hardware_acceleration {
				vendor = "nvidia|any"
			}
			network = true
		}
	}

	stage "foo" {
		cpus = 2.0
		use "github.com/zinc-sig/contrib/gcc-compile" {
			on {
				event = "throw"
				action = "skip" // "continue"
			}

			options {
				files = ["main.cpp"]
				flags = []
			}
		}
	}

	stage "baz" {
		after = ["foo"]
		use "github.com/zinc-sig/contrib/stdio-test" {
			options {
				executable = "a.out"
			}
			scenarios {
				defaults {
					visibility {
						// no-op effect
						filter {
							effect = "none"
							selector = [ for s in scenarios: s.exitCode == 0 ]
						}
						filter {
							effect = "hide"
							until = "2025-01-01T00:00:00+0800"
							selector = scenarios[*].stdout  // TBD
						}
						filter {
							effect = "hide"
							after = "{FINAL_GRADING}"  // replace with policy.deadline
							selector = "stdout|stderr"  // TBD
						}
					}
				}

				match {
					stdin = EOT<<
						1 2 3
					EOT
					args = ["1"]
					expected = EOT<<
						2 3 4
					EOT
					condition = STDOUT == EXPECTED
					score = 100
				}
				match "stdout-is-same" {
					condition = STDOUT == EXPECTED
					score = 100
					export = true
				}
			}
		}
	}

	stage "valgrind" {
		use "github.com/zinc-sig/contrib/valgrind" {
			options {
			}
			scenarios {
				match {
					condition = stage["baz"].outputs["stdout-is-same"].condition && LEAK == nil
					score = 100
				}
			}
		}
	}
}
```

## Future Extensions

- Version pinning for `use` dependencies