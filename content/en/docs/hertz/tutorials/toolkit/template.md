---
Title: "hz custom template use" 
Date: 2022-07-13
Weight: 2 
Description: >
---

Hertz provides command-line tools (hz) that support custom template features, including:
- Customize the layout template (i.e., the directory structure of the generated code)
- Custom package templates (i.e. service-related code structures, including handler, router, etc.)

Users can provide their own templates and rendering parameters, combined with the ability of hz, to complete the custom code generation structure.

# Custom layout template

> Users can modify or rewrite according to the default template to meet their own needs

Hz takes advantage of the "go template" capability to support defining templates in "yaml" format and uses "json" format to define rendering data.

The so-called layout template refers to the structure of the entire project. These structures have nothing to do with the specific idl definition, and can be directly generated without idl. The default structure is as follows:

```
.
├── biz
│   ├── handler
│   │   └── ping.go
│   │   └── ****.go               // Set of handlers divided by service, the position can be changed according to handler_dir 
│   ├── model
│   │   └── model.go              // idl generated struct, the position can be changed according to model_dir 
│   └── router //undeveloped custom dir 
│        └── register.go          // Route registration, used to call the specific route registration 
│             └── route.go        // Specific route registration location 
│             └── middleware.go   // Default middleware build location
├── .hz                           // hz Create code flags 
├── go.mod
├── main.go                       // Start the entrance 
├── router.go                     // User-defined route write location 
└── router_gen.go                 // hz generated route registration call
```

## IDL

```
// hello.thrift
namespace go hello.example

struct HelloReq {
    1: string Name (api.query="name");
}

struct HelloResp {
    1: string RespBody;
}


service HelloService {
    HelloResp HelloMethod(1: HelloReq request) (api.get="/hello");
}
```

## Command

```
hz new --mod=github.com/hertz/hello --idl=./hertzDemo/hello.thrift --customize_layout=template/layout.yaml:template/data.json
```

## The meaning of the default layout template

> Note: The following bodies are all go templates

```
layouts:
  # The directory of the generated handler will only be generated if there are files in the directory 
  - path: biz/handler/
    delims:
      - ""
      - ""
    body: ""
  # The directory of the generated model will only be generated if there are files in the directory 
  - path: biz/model/
    delims:
      - ""
      - ""
    body: ""
  # project main file,
  - path: main.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator.\n\npackage main\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n)\n\nfunc
    main() {\n\th := server.Default()\n\n\tregister(h)\n\th.Spin()\n}\n"
  # go.mod file, need template rendering data {{.GoModule}} to generate 
  - path: go.mod
    delims:
      - '{{'
      - '}}'
    body: "module {{.GoModule}}\n{{- if .UseApacheThrift}}\nreplace github.com/apache/thrift
    => github.com/apache/thrift v0.13.0\n{{- end}}\n"
  # .gitignore file
  - path: .gitignore
    delims:
      - ""
      - ""
    body: "*.o\n*.a\n*.so\n_obj\n_test\n*.[568vq]\n[568vq].out\n*.cgo1.go\n*.cgo2.c\n_cgo_defun.c\n_cgo_gotypes.go\n_cgo_export.*\n_testmain.go\n*.exe\n*.exe~\n*.test\n*.prof\n*.rar\n*.zip\n*.gz\n*.psd\n*.bmd\n*.cfg\n*.pptx\n*.log\n*nohup.out\n*settings.pyc\n*.sublime-project\n*.sublime-workspace\n!.gitkeep\n.DS_Store\n/.idea\n/.vscode\n/output\n*.local.yml\ndumped_hertz_remote_config.json\n\t\t
    \ "
  # .hz file, containing hz version, is the logo of the project created by hz, no need to transfer rendering data
  - path: .hz
    delims:
      - '{{'
      - '}}'
    body: "
      // Code generated by hz. DO NOT EDIT.

      hz version: v{{.hzVersion}}"
  # ping comes with ping handler 
  - path: biz/handler/ping.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator.\n\npackage handler\n\nimport (\n\t\"context\"\n\n\t\"github.com/cloudwego/hertz/pkg/app\"\n\t\"github.com/cloudwego/hertz/pkg/common/utils\"\n)\n\n//
    Ping .\nfunc Ping(ctx context.Context, c *app.RequestContext) {\n\tc.JSON(200,
    utils.H{\n\t\t\"message\": \"pong\",\n\t})\n}\n"
  # `router_gen.go` is the file that defines the route registration 
  - path: router_gen.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage main\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n\trouter
    \"{{.RouterPkgPath}}\"\n)\n\n// register registers all routers.\nfunc register(r
    *server.Hertz) {\n\n\trouter.GeneratedRegister(r)\n\n\tcustomizedRegister(r)\n\n}\n"
  # Custom route registration file 
  - path: router.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator.\n\npackage main\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n\thandler
    \"{{.GoModule}}/biz/handler\"\n)\n\n// customizeRegister registers customize routers.\nfunc
    customizedRegister(r *server.Hertz){\n\tr.GET(\"/ping\", handler.Ping)\n\n\t//
    your code ...\n}\n"
  # Default route registration file, do not modify it
  - path: biz/router/register.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage router\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n)
    \n\n// GeneratedRegister registers routers generated by IDL.\n
    func GeneratedRegister(r *server.Hertz) {\n\t//INSERT_POINT: DO NOT DELETE THIS LINE!\n}\n"
```


## The meaning of template rendering parameter file

When a custom template and render data are specified, the options specified on the command line will not be used as render data, so the render data in the template needs to be defined by the user.

Hz uses "json" to specify render data, as described below
```
{
  // global render parameters 
  "*": {
    "GoModule": "github.com/hz/test", // must be consistent with the command line, otherwise the subsequent generation of model, handler and other code will use the mod specified by the command line, resulting in inconsistency. 
    "ServiceName": "p.s.m", // as specified on the command line 
    "UseApacheThrift": false // Set "true"/"false" depending on whether to use "thrift" 
  },
  // router_gen.go route the registered render data, 
  // "biz/router"points to the module of the routing code registered by the default idl, do not modify it 
  "router_gen.go": {
    "RouterPkgPath": "github.com/hz/test/biz/router"
  }
}
```

## Customize a layout template

> At present, the project layout generated by hz is already the most basic skeleton of a hertz project, so it is not recommended to delete the files in the existing template.
>
> However, if the user wants a different layout, of course, you can also delete the corresponding file according to your own needs (except "biz/register.go", the rest can be modified)
>
> We welcome users to contribute their own templates

Assuming that the user only wants "main.go" and "go.mod" files, then we modify the default template as follows:

### template:

```
// layout.yaml
layouts:
  # project main file,
  - path: main.go
    delims:
      - ""
      - ""
    body: "// This is my customized hertz layout.\n\n
    package main
    \n\n
    import (\n
    \t\"github.com/cloudwego/hertz/pkg/app/server\"\n
    \t\"{{.GoModule}}/biz/router\"\n
    )\n\n
    func main() {\n
      \th := server.Default()\n\n
      \trouter.GeneratedRegister(h)\n\n
      \t// do what you wanted\n
      \t// add some render data: {{.MainData}}\n
      \th.Spin()\n
    }\n"
  # go.mod file, need template rendering data {{.GoModule}} to generate
  - path: go.mod
    delims:
      - '{{'
      - '}}'
    body: "module {{.GoModule}}\n{{- if .UseApacheThrift}}\nreplace github.com/apache/thrift
    => github.com/apache/thrift v0.13.0\n{{- end}}\n"
  # Default route registration file, no need to modify
  - path: biz/router/register.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage router\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n)
    \n\n// GeneratedRegister registers routers generated by IDL.\n
    func GeneratedRegister(r *server.Hertz) {\n\t//INSERT_POINT: DO NOT DELETE THIS LINE!\n}\n"
```

### render data:

```
{
  "*": {
    "GoModule": "github.com/hertz/hello",
    "ServiceName": "hello",
    "UseApacheThrift": true
  },
  "main.go": {
    "MainData": "this is customized render data"
  }
}
```
Command:
```
hz new --mod=github.com/hertz/hello --idl=./hertzDemo/hello.thrift --customize_layout=template/layout.yaml:template/data.json
```

# Custom package template

> The template address of the hz template:
>
> Users can modify or rewrite according to the default template to meet their own needs

-   The so-called package template refers to the code related to the idl definition. This part of the code involves the service, go_package/namespace, etc. specified when defining idl. It mainly includes the following parts:
-   handler.go: Handling function logic
-   router.go: the route registration logic of the service defined by the specific idl
-   register.go: logic for calling the content in `router.go`
-   ~~Model code: generated go struct; however, since the tool that uses plugins to generate model code currently does not have permission to modify the model template, this part of the function is not open for now~~

## Command

```
# After that, the package template rendering data will be provided, so the form of "k-v" is retained when entering the command, and ":" needs to be added after customize_package. 
hz new --mod=github.com/hertz/hello --handler_dir=handler_test --idl=hertzDemo/hello.thrift --customize_package=template/package.yaml:
```

## Default package template

Note: Custom package templates do not provide the ability to render data.

```
# The following data is obtained by yaml marshal, so it may look messy 
layouts:
  # Path only represents handler.go template, the specific handler path is determined by the default path and handler_dir 
  - path: handler.go
    delims:
      - '{{'
      - '}}'
    body: "// Code generated by hertz generator.\n\npackage {{.PackageName}}\n\nimport (\n\t\"context\"\n\n\t\"github.com/cloudwego/hertz/pkg/app\"\n\n
    {{- range $k, $v := .Imports}}\n\t{{$k}} \"{{$v.Package}}\"\n{{- end}}\n)\n\n{{range $_, $MethodInfo := .Methods}}\n{{$MethodInfo.Comment}}\n
    func {{$MethodInfo.Name}}(ctx context.Context, c *app.RequestContext) { \n\tvar err error\n\t{{if ne $MethodInfo.RequestTypeName \"\" -}}\n\t
    var req {{$MethodInfo.RequestTypeName}}\n\terr = c.BindAndValidate(&req)\n\tif err != nil {\n\t\tc.String(400, err.Error())\n\t\treturn\n\t}\n\t{{end}}\n
    \tresp := new({{$MethodInfo.ReturnTypeName}})\n\n\tc.{{.Serializer}}(200, resp)\n}\n{{end}}\n\t\t\t"
  # Path only represents the router.go template, whose path is fixed at: biz/router/
  - path: router.go
    delims:
      - '{{'
      - '}}'
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage {{$.PackageName}}\n\nimport
  (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n\n\t{{range $k, $v := .HandlerPackages}}{{$k}}
  \"{{$v}}\"{{end}}\n)\n\n/*\n This file will register all the routes of the services
  in the master idl.\n And it will update automatically when you use the \"update\"
  command for the idl.\n So don't modify the contents of the file, or your code will
  be deleted when it is updated.\n */\n\n{{define \"g\"}}\n{{- if eq .Path \"/\"}}r\n{{-
  else}}{{.GroupName}}{{end}}\n{{- end}}\n\n{{define \"G\"}}\n{{- if ne .Handler \"\"}}\n\t{{-
  .GroupName}}.{{.HttpMethod}}(\"{{.Path}}\", append({{.MiddleWare}}Mw(), {{.Handler}})...)\n{{-
  end}}\n{{- if ne (len .Children) 0}}\n{{.MiddleWare}} := {{template \"g\" .}}.Group(\"{{.Path}}\",
  {{.MiddleWare}}Mw()...)\n{{- end}}\n{{- range $_, $router := .Children}}\n{{- if
  ne .Handler \"\"}}\n\t{{template \"G\" $router}}\n{{- else}}\n\t{\t{{template \"G\"
  $router}}\n\t}\n{{- end}}\n{{- end}}\n{{- end}}\n\n// Register register routes based
  on the IDL 'api.${HTTP Method}' annotation.\nfunc Register(r *server.Hertz) {\n{{template
  \"G\" .Router}}\n}"
  # path only represents register.go template, the path of register is fixed to biz/router/register.go
  - path: register.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage router\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n\t{{$.PkgAlias}}
  \"{{$.Pkg}}\"\n)\n\n// GeneratedRegister registers routers generated by IDL.\nfunc
  GeneratedRegister(r *server.Hertz){\n\t//INSERT_POINT: DO NOT DELETE THIS LINE!\n\t{{$.PkgAlias}}.Register(r)\n}\n"
  - path: model.go
    delims:
      - ""
      - ""
    body: ""
  # path only represents middleware.go template, the path of middleware is the same as router.go: biz/router/namespace/ 
  - path: middleware.go
    delims:
      - '{{'
      - '}}'
    body: "// Code generated by hertz generator.\n\npackage {{$.PackageName}}\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app\"\n)\n\n{{define
  \"M\"}}\nfunc {{.MiddleWare}}Mw() []app.HandlerFunc {\n\t// your code...\n\treturn
  nil\n}\n{{range $_, $router := $.Children}}{{template \"M\" $router}}{{end}}\n{{-
  end}}\n\n{{template \"M\" .Router}}\n\t\t"
  # path only represents the template of the client.go, the generation path of the client code is specified by the user "${client_dir}"
- path: client.go
  delims:
    - '{{'
    - '}}'
  body: "// Code generated by hertasdasdsadaz generator.\n\npackage {{$.PackageName}}\n\nimport (\n
\   \"github.com/cloudwego/hertz/pkg/app/client\"\n\t\"github.com/cloudwego/hertz/pkg/common/config\"\n)\n\ntype
{{.ServiceName}}Client struct {\n\tclient * client.Client\n}\n\nfunc New{{.ServiceName}}Client(opt
...config.ClientOption) (*{{.ServiceName}}Client, error) {\n\tc, err := client.NewClient(opt...)\n\tif
err != nil {\n\t\treturn nil, err\n\t}\n\n\treturn &{{.ServiceName}}Client{\n\t\tclient:
c,\n\t}, nil\n}\n\t\t"
```

## Customize a package template

> Like layout templates, users can also customize package templates.
>
> As far as the templates provided by the package are concerned, the average user may only need to customize handler.go templates, because router.go/middleware.go/register.go are generally related to the idl definition and the user does not need to care, so hz currently also fixes the location of these templates, and generally does not need to be modified.
>
> Therefore, users can customize the generated handler template according to their own needs to speed up development; however, since the default handler template integrates some model information and package information, the hz tool is required to provide rendering data. This part of the user can modify it according to their own situation, and it is generally recommended to leave model information.

Here is an example of a simple custom handler template, adding some comments to the handler:

### template:

```
# The following data is obtained by yaml marshal, so it may look messy 
layouts:
  # Path only represents handler.go template, the specific handler path is determined by the default path and handler_dir 
  - path: handler.go
    delims:
      - '{{'
      - '}}'
    body: "// this is my custom handler.\n\npackage {{.PackageName}}\n\nimport (\n\t\"context\"\n\n\t\"github.com/cloudwego/hertz/pkg/app\"\n\n
    {{- range $k, $v := .Imports}}\n\t{{$k}} \"{{$v.Package}}\"\n{{- end}}\n)\n\n{{range $_, $MethodInfo := .Methods}}\n{{$MethodInfo.Comment}}\n
    func {{$MethodInfo.Name}}(ctx context.Context, c *app.RequestContext) { \n
    \t//  you can code something \n\tvar err error\n\t{{if ne $MethodInfo.RequestTypeName \"\" -}}\n\t
    var req {{$MethodInfo.RequestTypeName}}\n\terr = c.BindAndValidate(&req)\n\tif err != nil {\n\t\tc.String(400, err.Error())\n\t\treturn\n\t}\n\t{{end}}\n
    \tresp := new({{$MethodInfo.ReturnTypeName}})\n\n\tc.{{.Serializer}}(200, resp)\n}\n{{end}}\n\t\t\t"
  # path only represents router.go template, its path is fixed at: biz/router/namespace/ 
  - path: router.go
    delims:
      - '{{'
      - '}}'
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage {{$.PackageName}}\n\nimport
  (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n\n\t{{range $k, $v := .HandlerPackages}}{{$k}}
  \"{{$v}}\"{{end}}\n)\n\n/*\n This file will register all the routes of the services
  in the master idl.\n And it will update automatically when you use the \"update\"
  command for the idl.\n So don't modify the contents of the file, or your code will
  be deleted when it is updated.\n */\n\n{{define \"g\"}}\n{{- if eq .Path \"/\"}}r\n{{-
  else}}{{.GroupName}}{{end}}\n{{- end}}\n\n{{define \"G\"}}\n{{- if ne .Handler \"\"}}\n\t{{-
  .GroupName}}.{{.HttpMethod}}(\"{{.Path}}\", append({{.MiddleWare}}Mw(), {{.Handler}})...)\n{{-
  end}}\n{{- if ne (len .Children) 0}}\n{{.MiddleWare}} := {{template \"g\" .}}.Group(\"{{.Path}}\",
  {{.MiddleWare}}Mw()...)\n{{- end}}\n{{- range $_, $router := .Children}}\n{{- if
  ne .Handler \"\"}}\n\t{{template \"G\" $router}}\n{{- else}}\n\t{\t{{template \"G\"
  $router}}\n\t}\n{{- end}}\n{{- end}}\n{{- end}}\n\n// Register register routes based
  on the IDL 'api.${HTTP Method}' annotation.\nfunc Register(r *server.Hertz) {\n{{template
  \"G\" .Router}}\n}"
  # path only represents register.go template, the path of register is fixed to biz/router/register.go
  - path: register.go
    delims:
      - ""
      - ""
    body: "// Code generated by hertz generator. DO NOT EDIT.\n\npackage router\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app/server\"\n\t{{$.PkgAlias}}
  \"{{$.Pkg}}\"\n)\n\n// GeneratedRegister registers routers generated by IDL.\nfunc
  GeneratedRegister(r *server.Hertz){\n\t//INSERT_POINT: DO NOT DELETE THIS LINE!\n\t{{$.PkgAlias}}.Register(r)\n}\n"
  - path: model.go
    delims:
      - ""
      - ""
    body: ""
  # path only represents middleware.go template, the path of middleware is the same as router.go: biz/router/namespace/ 
  - path: middleware.go
    delims:
      - '{{'
      - '}}'
    body: "// Code generated by hertz generator.\n\npackage {{$.PackageName}}\n\nimport (\n\t\"github.com/cloudwego/hertz/pkg/app\"\n)\n\n{{define
  \"M\"}}\nfunc {{.MiddleWare}}Mw() []app.HandlerFunc {\n\t// your code...\n\treturn
  nil\n}\n{{range $_, $router := $.Children}}{{template \"M\" $router}}{{end}}\n{{-
  end}}\n\n{{template \"M\" .Router}}\n\t\t"
  # path only represents the template of the client.go, the generation path of the client code is specified by the user "${client_dir}" 
- path: client.go
  delims:
    - '{{'
    - '}}'
  body: "// Code generated by hertz generator.\n\npackage {{$.PackageName}}\n\nimport (\n
\   \"github.com/cloudwego/hertz/pkg/app/client\"\n\t\"github.com/cloudwego/hertz/pkg/common/config\"\n)\n\ntype
{{.ServiceName}}Client struct {\n\tclient * client.Client\n}\n\nfunc New{{.ServiceName}}Client(opt
...config.ClientOption) (*{{.ServiceName}}Client, error) {\n\tc, err := client.NewClient(opt...)\n\tif
err != nil {\n\t\treturn nil, err\n\t}\n\n\treturn &{{.ServiceName}}Client{\n\t\tclient:
c,\n\t}, nil\n}\n\t\t"
```
Command:
```
# After that, the package template rendering data will be provided, so the form of "k-v" is retained when entering the command, and ":" needs to be added after customize_package. 
hz new --mod=github.com/hertz/hello --handler_dir=handler_test --idl=hertzDemo/hello.thrift --customize_package=template/package.yaml:
```

The handler is generated as follows:
```
// this is my custom handler.

package example

import (
	"context"

	"github.com/cloudwego/hertz/pkg/app"
	example "test/test2/biz/model/hello/example"
)

// HelloMethod .
// @router /hello [GET]
func HelloMethod(ctx context.Context, c *app.RequestContext) {
	//  you can code something
	var err error
	var req example.HelloReq
	err = c.BindAndValidate(&req)
	if err != nil {
		c.String(400, err.Error())
		return
	}

	resp := new(example.HelloResp)

	c.JSON(200, resp)
}

// OtherMethod .
// @router /other [POST]
func OtherMethod(ctx context.Context, c *app.RequestContext) {
	//  you can code something
	var err error
	var req example.OtherReq
	err = c.BindAndValidate(&req)
	if err != nil {
		c.String(400, err.Error())
		return
	}

	resp := new(example.OtherResp)

	c.JSON(200, resp)
}

```

# Precautions

## Precautions for using layout templates

When the user uses the layout custom template, the generated layout and rendering data are taken over by the user, so the user needs to provide the rendering data of the defined layout.

## Precautions for using package templates

Generally speaking, when users use package templates, most of them are to modify the default handler template; however, hz does not provide a single handler template at present, so when updating an existing handler file, the default handler template will be used to append a new handler function to the end of the handler file. When the corresponding handler file does not exist, a custom template will be used to generate the handler file.