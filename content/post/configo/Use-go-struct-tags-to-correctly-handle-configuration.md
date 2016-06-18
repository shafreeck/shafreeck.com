+++
author = "Shafreeck Sea"
date = "2016-06-18T17:35:36+08:00"
title = "Use go struct tags to correctly handle configuration"

+++

[configo](https://github.com/shafreeck/configo) is a `toml` parsing library focus on configuration. This article describes what I need to parse a configuration file, why I created configo and how to use it to keep consistent between your code and your configuration file. It is a young project and under active development, feel free to feedback or send me pull request if there is anything you want to improve.

## What I want to correctly handle a configuration file

I have started several projects these days. They all need configuration files to configure its server options.

I looked several configuration file format including `json`, `ini`, `yaml`, and `toml`. Finally I decide to use `toml` which is designed specifically for configuration file and supplies many features including supporting multiple sections(tables), adding comment inside an array and so on. See the [docs](https://github.com/toml-lang/toml#comparison-with-other-formats) for information about comparing toml with others.

All my projects are written in golang. So I need a library to load the toml configuration file. There are several awesome projects can do this, but they all miss something I really want.

## 1. Loading `TOML` in a gopher way

JSON(encoding/json) is a good example to explain what I mean "in a gopher way" here. Go has a very simple api to parse a `json` file.

```go
json.Unmarshal(data []byte, v interface{}) error
```

You define a go struct according to the `json` and json.Unmarshal it then the struct object's fields will be filled using the values in `json`.

What I really want is a `toml.Unmarshal` method to `toml`. In fact many projects already support this, but they all miss some semantics to handle configuration.

## 2. Configuration semantics

The standard library `encoding/json` is designed to serialize and deserialize json data. However, configuration is different in many aspects. First, configuration is usually used in almost all services which power your company(maybe facebook, twitter or something else). An error in configuration file may cause a disaster to your service. Second, configuration files are usually created and managed by operators or developers, mistakes by humans can be nearly unavoidable. Third, not all configuration options have to be configured, some is required and some can be optional with a default value.

So the core features I need should be supplied is as below:

* Report errors when the option configured is unknown or when something required is missing.
* Validate the value if the option can be recognized.
* A configuration key can be required or optional, and we can supply a default value when it is optional.
* Write the go struct and generate the configuration file base on it.

## 3. Satisfy all things above in a simple way

Go support strut tags which can be useful when marshal or unmarshal a go struct. It is a key value format string wrapped around by backticks.

```go
type Config struct {
  Listen string `json:"listen,omitempty"`
}
```
Struct tags is cool, it should be more powerful than what you had see.

Configo leverage go struct tags to supply a very simple interface to marshal and unmarshal `TOML` with all these requirements above being satisfied. The tags can be defined as this `cfg:"name, default value or required, validating rules, description"`

For example:

```go
type Config struct {
  Listen string `cfg:"listen, required, netaddr, The listen address of server"`
}
```

`listen` is the key mapped in the configuration file. `required` means this should be explicitly configured in the file, or you can specify a valid address for example `:8804` as default value. `netaddr` is the validation rule configo used to validate the value of what you configured. The last part is a description of the key. It is used as a comment when generate a toml based on the go struct.

## Introduction to configo

Configo use [shafreeck/toml](https://github.com/shafreeck/toml) witch is forked from [naoina/toml](https://github.com/naoina/toml) to marshal and unmarshal the toml data. It is modified to parse the tags configo used and generate toml with more human friendly information. Do not use it if you just want a toml parser.

[Govalidator](https://github.com/asaskevich/govalidator) is used in configo to validate configured values. It has
many kinds of validation rules to satisfy your requirements and it is convenient to add your own rules. `netaddr` is the rule configo self added to verify a legal listen address.

The tag key of configo is `cfg`. It obeys the rule of go strut tag format which is key:"value string". There should be no spaces between key and value string. The value string configo used can be splited into four parts with commas. All part can be empty but the comma should not be omitted.

Let's try to walk through the flows using the main features of configo.

### Define a struct with configo tag
```go
type Config struct {
  Listen string `cfg:"listen, :8804, netaddr, The address the server to listen"`
  MaxConn int `cfg:"max-connection, 10000, numeric, Max number of concurrent connections"`
  Redis struct{
    Cluster []string `cfg:"cluster, required, dialstring", The addresses of redis cluster`
  }
}
```

You can see that all the configuration related things are declared using strut tags. Combining the strut and how it should be configured in one place makes it more clear to developers.

### Generate a toml file
```go
func main() {
    c := &Config{}
    if data, err := configo.Marshal(c); err != nil {
        fmt.Println(err)
    } else {
        fmt.Printf("%s", data)
    }
}
```

The complete code can be run is as follows

```go
package main

import (
    "fmt"
    "github.com/shafreeck/configo"
)

type Config struct {
    Listen  string `cfg:"listen, :8804, netaddr, The address the server to listen"`
    MaxConn int    `cfg:"max-connection, 10000, numeric, Max number of concurrent connections"`
    Redis   struct {
        Cluster []string `cfg:"cluster, required, dialstring", The addresses of redis cluster`
    }
}

func main() {
    c := &Config{}
    if data, err := configo.Marshal(c); err != nil {
        fmt.Println(err)
    } else {
        fmt.Printf("%s", data)
    }
}
```

Name this source file as `conf.go` and run
```sh
go run conf.go > conf.toml
```

The generated file should be like this:

```toml
#type:        string
#rules:       netaddr
#description: The address the server to listen
#default:     :8804
#listen=""

#type:        int
#rules:       numeric
#description: Max number of concurrent connections
#default:     10000
#max-connection=0

[redis]
#type:        []string
#rules:       dialstring
#required
cluster=[]
```

Yes, configo not just marshal the go object into toml data, it also adds rich useful information about the configuration option, including what the type of the value is and what rules(you can use more than one rule split by spaces) you should obey. Also there is a description describes more details if you supplied. The option will be commented out if there is a default value specified in struct tag. The default value is also put there to remind the developers or operators. The configuration key that is required or missing a default value is uncommented and set with its golang empty value. You should configure it with legal value that match the validation rules.

### Load the toml file as configuration

Edit `conf.toml` and configure redis cluster

```toml
cluster = ["127.0.0.1:6379", "127.0.0.1:6380"]
```

Load the toml file using configo.Unmarshal

```go
package main

import (
    "io/ioutil"
    "log"

    "github.com/shafreeck/configo"
)

type Config struct {
    Listen  string `cfg:"listen, :8804, netaddr, The address the server to listen"`
    MaxConn int    `cfg:"max-connection, 10000, numeric, Max number of concurrent connections"`
    Redis   struct {
        Cluster []string `cfg:"cluster, required, dialstring", The addresses of redis cluster`
    }
}

func main() {
    data, err := ioutil.ReadFile("conf.toml")
    if err != nil {
        if err != nil {
            log.Fatalln(err)
        }
    }

    c := &Config{}

    if err := configo.Unmarshal(data, c); err != nil {
        log.Fatalln(err)
    }

    log.Println(c)
}
```

Then

```sh
go run load.go
```

The output should be

```
2016/06/18 19:12:35 &{:8804 10000 {[127.0.0.1:6379 127.0.0.1:6380]}}
```

You can see that though we commented out listen and max-connection, they are all set using default value after loaded.

## Summary
Configo leverage the go strut tags to supply a simple and powerful api to parse and generate a toml configuration file. Generating toml from your source code makes it simple be consistent between your conf and code. Use default value to minify your configuration or set "required" to ensure the important things have been configured. At last, don't forget validation is really important to avoid mistakes.
