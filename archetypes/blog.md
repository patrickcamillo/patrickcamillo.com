+++
title = "{{ replace .Name "-" " " | title }}"
date = "{{ .Date }}"
# atualizado = ""
draft = true
toc = false
comments = true
fediverse = 0
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = [{{ range $plural, $terms := .Site.Taxonomies }}{{ range $term, $val := $terms }}"{{ printf "%s" $term }}",{{ end }}{{ end }}]
+++

This is a page about »{{ replace .Name "-" " " | title }}«.
