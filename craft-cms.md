# Craft CMS Image CDN snippets

Put `img.twig` into your templates directory.

## Usage

```twig
{% include 'components/img' with {
  image: entry.image.one(),
  widths: [ 400, 600, 800, 1000 ],
  sizes: '100vw'
} %}
```

## `img.twig`

```twig
{%- comment -%}
	IMAGE SNIPPET
	===============================
	Takes an image object and a few parameters and outputs an <img> tag.
	General parameters:
		- img       	— The image object as provided by Craft. Alternatively provided as 'img_obj'.
    - picClass    - A string of classes to apply to the picture tag
		- class     	— A string of classes to apply to the img tag
		- alt       	— Custom alt text, defaults to an 'alt' field on the image
		- loading   	— Native lazy-loading, set to 'eager' for high-priority images. Defaults to 'lazy'
		- decoding  	— Sets 'decoding' attribute, likely not necessary to change. Defaults to 'async'
		- attributes 	— Adds misc attributes to img tag

	Image sizing parameters:
		- widths  — Comma-seperated string in the form '300,400,500' specifying what image sizes to use
						in the srcset attribute. The last (or only) one will be used as the 'src' attribute.
						Use in conjunction with 'sizes' for responsive images.
						Defaults to '400,600,900,1200,1600'
		- sizes   — Sets the 'sizes' attribute of the img tag, used to improve the performance of
						responsive images. Defaults to '(min-width: 87rem) 85rem, calc(100vw - 2rem)'
		- ratio   — Crops images to a certain aspect ratio. Provide a decimal greater than 0, calculated
						by width / height. For example to crop to a 4:3 ratio provide `1.3`, or `1` for a square.
						Defaults to the image's intrinsic aspect ratio, so no cropping.
{%- endcomment -%}
{%- if image -%}
	{%- set picClass = picClass ?? false -%}
	{%- set class = class ?? false -%}
	{%- set sizes = sizes | default('(min-width: 82rem) 80rem, calc(100vw - 2vw)') -%}
	{%- set alt = alt ?? image.alt ?? '' -%}
	{%- set loading = loading ?? 'lazy' -%}
	{%- set decoding = decoding ?? 'async' -%}
	{%- set attributes = attributes ?? '' -%}

	{%- set widths = widths ?? [ 300, 450, 600, 800, 1000, 1200 ] -%}
	{%- set largestWidth = width ?? widths | last -%}
	{%- set fallbackWidth = fallbackWidth ?? largestWidth -%}

	{%- set ratio = (ratio ?? image.width / image.height) | round(2)  -%}

	<picture {% if picClass %}class="{{ picClass }}"{% endif %}>
		<source
			media="(-webkit-min-device-pixel-ratio: 1.5)"
			srcset="{{ _self.srcset(image.url, widths, ratio, 40, 2) }}"
			sizes="{{ sizes }}"
		>
		<img
			{% if class %}class="{{ class }}"{% endif %}
			src="{{ _self.src(image.url, fallbackWidth, ratio) }}"
			srcset="{{ _self.srcset(image.url, widths, ratio) }}"
			sizes="{{ sizes }}"
			width="{{ largestWidth }}"
			height="{{ (largestWidth / ratio) | round(2) }}"
			alt="{{ alt }}"
			{% if loading != false %}loading="{{ loading }}"{% endif %}}}
			{% if decoding != false %}decoding="{{ decoding }}"{% endif %}
			{{ attributes }}
		>
	</picture>
{%- endif -%}

{%- macro src (url, width, ratio, quality = 80, scale = 1) -%}
	{{ url }}?auto=format,compress&w={{ width * scale }}&ar={{ ratio }}:1&q={{ quality }}
{%- endmacro -%}

{%- macro srcset (url, widths, ratio, quality = 80, scale = 1) -%}
	{%- for width in widths -%}
		{{ _self.src(url, width, ratio, quality, scale) }} {{ width * scale }}w,
	{%- endfor -%}
{%- endmacro -%}
```
