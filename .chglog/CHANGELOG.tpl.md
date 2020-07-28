{{ range .Versions }}
## Changelog

{{ range .CommitGroups -}}
{{ range .Commits -}}
- [`{{ .Hash.Short }}`](https://github.com/spinnaker/kustomization-base/commit/{{ .Hash.Long }}): {{ .Header }}{{ if .Merge }}({{ .Merge.Ref }}){{ end }}
{{ end -}}
{{ end -}}
{{ end -}}
