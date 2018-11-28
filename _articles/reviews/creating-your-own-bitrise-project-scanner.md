---
title: Creating your own Bitrise project scanner
date: 2018-11-27 14:30:32 +0000
redirect_from: []
published: false

---
The project scanner is a tool that identifies the given project's type and generates a basic Bitrise configuration. Each supported project type has its own scanner: these scanners are stored as separate packages.

When adding a new project on the website or initializing a project on your own machine, the [bitrise-init](https://github.com/bitrise-core/bitrise-init) tool iterates through every scanner, calls the scanner interface methods on each of them and collects their outputs. Based on these outputs, a basic configuration is generated.

The possible workflows are described in a scan result model. The model consists of:

* options
* configs
* warnings

Here is the basic structure of the model, in YAML:

```yaml
options:
  DETECTED_PLATFORM_1: OptionModel
  DETECTED_PLATFORM_2: OptionModel
  ...

configs:
  DETECTED_PLATFORM_1:
    CONFIG_NAME_1: ConfigModel
    CONFIG_NAME_2: ConfigModel
    ...
  DETECTED_PLATFORM_2:
    CONFIG_NAME_1: ConfigModel
    CONFIG_NAME_2: ConfigModel
    ...
  ...

warnings:
  DETECTED_PLATFORM_1:
  - "warning message 1"
  - "warning message 2"
  ...
  DETECTED_PLATFORM_2:
  - "warning message 1"
  - "warning message 2"
  ...
```

* Every platform scanner writes its possible options, configurations and warnings into this model. These will be translated into step input values by choosing the desired values for the given options.
* Every option chain’s last option selects a configuration.
* Warnings display the issues with the given project setup.

## Options

`Options` represents a question and the possible answers to the question. For example:

* Question: What is the path to the iOS project files?
* Possible answers: List of possible paths to check

These questions and answers are translated into step inputs. The scanner should either determine the input value or let the user select or type the value.

For example, the `Xcode Archive & Export for iOS` Step has an input called `export-method`. This informs the Step of the type of .ipa you want to export. The value cannot be determined based on the source code so the scanner collects every possible value and presents them to the user in the form of a list to choose from.

Selecting an option can start a chain: it can lead to different options being presented afterwards. For example, if you select an Xcode scheme that has associated test targets, it leads to different "questions". Similarly, selecting a certain option can lead to a different workflow being generated afterwards.

### The option model

The OptionModel represents an input option. It looks like this in Go:

```Go
// OptionModel ...
type OptionModel struct {
    Title  string
    EnvKey string

    ChildOptionMap map[string]*OptionModel
    Config         string
}
```

* `Title`: the human readable name of the input
* `EnvKey`: it represents the input's key in the step model
* `ChildOptionMap`: the map of the subsequent options if the user chooses a given value for the option

For example, let's see a scenario where you choose a value for the `Scheme` input. You will have a `value_map` in the `options`. The possible values are:

* `SchemeWithTest`
* `SchemeWithoutTest`

By choosing `SchemeWithTest`, the next option will be related to the simulator used to perform the test.

By choosing `SchemeWithoutTest`, the next option will be about the export method for the .ipa file.

{% raw %}

```JSON
{
    "title": "Scheme",
    "env_key": "scheme",
    "value_map": {
        "SchemeWithTest": {
            "title": "Simulator name",
            "env_key": "simulator_name",
            ...
        },
        "SchemeWithoutTest": {
            "title": "Export method",
            "env_key": "export_method",
            ...
        }
    }
}
```

{% endraw %}

Every option chain has a first option: this is called `head`. The possible values of the options can branch the option chain.

Every option branch's last `options` must have a `config` property set. `config` holds the id of the generated Bitrise configuration which will select the values for the options of the given project.

An options chain's last `options` cannot have a `value_map`.

{% raw %}

```JSON
{
    "title": "Scheme",
    "env_key": "scheme",
    "value_map": {
        "SchemeWithTest": {
            "title": "Simulator name",
            "env_key": "simulator_name",
            "value_map": {
                "-": {
                    "config": "bitrise_config_with_test",
                }
            }
        },
        "SchemeWithoutTest": {
            "title": "Export method",
            "env_key": "export_method",
            "value_map": {
                "development": {
                    "config": "bitrise_config_without_test",
                },
                "app-store": {
                    "config": "bitrise_config_without_test",
                },
                "ad-hoc": {
                    "config": "bitrise_config_without_test",
                }
            }
        }
    }
}
```

{% endraw %}

## Scanners

Scanners generate the possible `options` chains and the possible workflows for the `options` per project type. The `ActiveScanner` variable holds each scanner implementation. Every specific scanner implements the `ScannerInterface` method.

```Go
// ScannerInterface ...
type ScannerInterface interface {
    Name() string
    DetectPlatform(string) (bool, error)

    Options() (models.OptionModel, models.Warnings, error)
    Configs() (models.BitriseConfigMap, error)

    DefaultOptions() models.OptionModel
    DefaultConfigs() (models.BitriseConfigMap, error)

    ExcludedScannerNames() []string
}
```

* `Name() string`: it is used for logging and storing the scanner output (warnings, options and configs). The scanner output is stored in `map[SCANNER_NAME]OUTPUT`. For example, an `options` map for an iOS project is stored in `optionsMap[ios]options`. 
* `DetectPlatform(string) (bool, error)`: it is used to determine if the given search directory contains the project type or not.
* `Options() (models.OptionModel, models.Warnings, error)`: it is used to generate option branches for the project. Each branch should define a complete and valid option set to build the final bitrise config model. Every option branch’s last `Options` has to store a configuration id, which will be filled with the selected options.
* `Configs() (models.BitriseConfigMap, error)`: it is used to generate the possible configs. BitriseConfigMap’s each element is a bitrise config template which will be fulfilled with the user selected option values.
* `DefaultOptions() models.OptionModel and DefaultConfigs() (models.BitriseConfigMap, error)` : they are used to generate the options and configs without scanning the given project. In this case every required step input value is provided by the user. This way even if a scanner fails, the user has an option to get started.