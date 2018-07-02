# Best practices for developing Nextstrain pathogen builds

These are suggested best practices for developing modular augur pathogen builds
in per-pathogen repositories.  Original discussion of these recommendations
happened [in a GitHub
Gist](https://gist.github.com/tsibley/0e760b9f7d7955fc9d48ad87728a8af4).


## Use a config.yaml file

Configuration is data and should live inside a YAML file named `config.yaml`.
You can access it in your Snakefile by including the line:

    configfile: "config.yaml"

and then using the `config` dictionary provided in scope afterwards.


## Allow for easy local config overrides

By including the following snippet in your Snakefile, you can allow for
optional, local configuration overrides which are more persistent than using
`--config` options to `snakemake`.

    from pathlib import Path

    if Path("config_local.yaml").is_file():
        configfile: "config_local.yaml"

If you do this, you should also add a `/config_local.yaml` line to your
repository's top-level `.gitignore` file.


## Use Snakemake `params:` block to map into `config` dictionary

For example, do this:

    params:
        name = config["name"]
    shell:
        "echo {params.name:q}"

instead of using the `config` dictionary directly in the shell command.  This
has several benefits:

  * Interpolation of dictionary lookups in the shell commands is non-standard
    and confusing.  (You have to use `{config[name]}`, for example. Note that
    the dictionary key is unquoted.)

  * Param definitions can use arbitrary Python expressions, so you can do more
    complicated things than you can with direct interpolation, such as list
    comprehensions.

  * Snakemake can automatically discover which rules have parameter values that
    are different than the last run and show what output files are affected
    (`--list-params-changes`).


## Always use quoted (:q) interpolations

When building shell commands to run, Snakemake does not by default properly
quote interpolated values.  This works fine if the interpolated value doesn't
contain spaces or other special shell metacharacters (like quotes or
backslashes), but it is fragile and a time-bomb waiting to break on future
values.

Standard best practice in any language or environment is to always quote
parameters in generated shell commands.  Snakemake supports this using the `:q`
modifier for interpolation:

    params:
        file = "filename with spaces.txt"
    shell:
        "wc -l {params.file:q}"

Not quoting these values is also a security risk.

It may be tempting to make an exception for parameters with multiple values where
you want each become a separate command-line argument, such as a parameter listing
three filenames.  In this case, however, it's recommended that you make the parameter
a list instead of a single string.  Snakemake will interpolate it correctly:

    params:
        files = ["a.txt", "b.txt", "c.txt"]
    shell:
        "wc -l {params.files:q}"


## Use triple-quoted command definitions

Using triple-quoted (`"""` or `'''`) command definitions make it much easier to
build readable commands with one-option per line.  It also avoids any nested
quoting issues if you need to use literal single or double quotes in your
command.

Example:

    shell:
        """
        augur parse \
            --sequences {input:q} \
            --fields {params.fields:q} \
            --output-sequences {output.sequences:q} \
            --output-metadata {output.metadata:q}
        """


## Provide descriptive messages for each rule

Rules support a `message:` block which defines a string to print when the rule
is run.  This is very helpful for tracking build progress and in general
describing what's happening.  Note that you can use interpolation within the
message, so you can surface important parameter values or conditional inputs or
anything else that might be useful.

Example:

    rule traits:
        message: "Inferring ancestral traits {params.columns!s}"
