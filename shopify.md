# Shopify Image CDN snippets

Put into the `snippets/` directory within your theme.

## Usage

```liquid
{%- render 'img',
  img: section.settings.image,
  widths: '400,600,800,1000',
  sizes: '(min-width: 768px) 50vw, 100vw'
-%}
```

## `img.liquid`

```liquid
{%- comment -%}
	IMAGE SNIPPET
	===============================
	Takes an image object and a few parameters and outputs an <img> tag.
	General parameters:
		- img       	— The image object as provided by Shopify. Alternatively provided as 'img_obj'.
		- class     	— A string of classes to apply to the img tag. Alternatively provided as 'class_str'.
		- alt       	— Custom alt text, defaults to the alt text as set in Shopify
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

{%- comment -%}
	Set misc attributes
{%- endcomment -%}
{% assign img = img | default: img_obj %}
{% if img %}
	{% assign class = class | default: class_str %}
	{% assign loading = loading | default: 'lazy', allow_false: true %}
	{% assign decoding = decoding | default: 'async', allow_false: true %}
	{% assign attributes = attributes | default: '' %}
	{% assign altText = img.alt %}
	{% if alt != blank %}
		{% assign altText = alt %}
	{% endif %}

	{%- comment -%}
		Set up initial variables for image sizing and resizing
	{%- endcomment -%}
	{% assign sizes = sizes | default: '(min-width: 87rem) 85rem, calc(100vw - 2rem)', allow_false: true %}
	{% assign widths = widths | default: '400,600,900,1200,1600' %}
	{% assign ratio = ratio | default: img.aspect_ratio | plus: 0.0 %}

	{%- capture actualWidths -%}
		{% render 'img--widths',
			widths: widths,
			img: img,
			ratio: ratio
		%}
	{%- endcapture -%}

	{% assign widthArr = actualWidths | split: ',' %}

	{%- comment -%}
		Use the last item in the width array to build a fallback src attribute
	{%- endcomment -%}
	{% assign fallbackW = widthArr.last | strip %}
	{% assign fallbackH = fallbackW | divided_by: ratio | round %}
	{% assign src = img | image_url: width: fallbackW, height: fallbackH, crop: 'center' %}

	{%- if sizes and widthArr.size > 1 -%}
		{%- capture srcset -%}
			{%- render 'img--srcset',
				img: img,
				widthArr: widthArr,
				ratio: ratio
			-%}
		{%- endcapture -%}
	{%- endif -%}

	<img
		{% comment %} Misc attributes {% endcomment %}
		{% if class != blank %}
			class="{{ class }}"
		{% endif %}
		alt="{{ altText | escape }}"
		{% if loading != false %}loading="{{ loading }}"{% endif %}
		{% if decoding != false %}decoding="{{ decoding }}"{% endif %}
		{{ attributes }}

		{% comment %} image sizing {% endcomment %}
		src="{{ src }}"
		width="{{ fallbackW }}"
		height="{{ fallbackH }}"
		{% if srcset != '' %}
			srcset="{{ srcset | strip }}"
			sizes="{{ sizes }}"
		{% endif %}
	>
{% endif %}
```

## `img--srcset.liquid`

```liquid
{%- comment -%}
	If there's more than one width and sizes isn't false, turn the width array into a srcset attribute
{%- endcomment -%}
{%- capture srcset -%}{%- endcapture -%}
{%- for width in widthArr -%}
	{%- assign widthStrip = width | strip -%}
	{%- if widthStrip != blank -%}
		{%- assign widthNum = widthStrip | plus: 0 -%}
		{%- assign height = widthNum | divided_by: ratio -%}
		{%- capture srcset -%}
			{{ srcset }}
			{{ img | image_url: width: widthNum, height: height, crop: 'center' }} {{ widthNum | round }}w,
		{%- endcapture -%}
	{%- endif -%}
{%- endfor -%}
{{ srcset }}
```

## `img--width.liquid`

```liquid
{%- assign actualWidth = null -%}
{%- assign widthNum = width | plus: 0.0 %}
{%- assign heightNum = height | plus: 0.0 %}
{%- assign ratio = widthNum | divided_by: heightNum -%}
{%- if img.width > widthNum and img.height > heightNum -%}
	{%- assign actualWidth = widthNum -%}
{%- else -%}
	{%- assign actualWidth = img.width -%}
	{%- assign widthBasedOnHeight = img.height | times: ratio | round -%}
	{%- if widthBasedOnHeight < img.width -%}
		{%- assign actualWidth = widthBasedOnHeight -%}
	{%- endif -%}
{%- endif -%}
{{ actualWidth | round }}
```

## `img--widths.liquid`

```liquid
{%- comment -%}
	Convert theoretical width list into the array of image widths we'll actually use.
	Because Shopify won't crop an image if we specify dimensions larger that it's intrinsic width, we need to
		check that every width we use is <= the intrinsic width. That's what we do in this for loop.
	If a theoretical width is larger than the intrinsic, we set a property to stop processing more and add the
		image with it's intrinsic size to the end of the array.
	We then need to do some tidying up to convery this into an array we can loop over.
{%- endcomment -%}
{%- assign theoreticalWidthArr = widths | split: ',' -%}
{%- capture actualWidths -%}{%- endcapture -%}
{%- assign hasMaxedOut = false -%}
{%- for width in theoreticalWidthArr -%}
	{%- unless hasMaxedOut -%}
		{%- comment -%} We need to coerce the width string into a number to do comparisons {%- endcomment -%}
		{%- assign widthNum = width | plus: 0 -%}
		{%- assign heightNum = widthNum | divided_by: ratio | round -%}
		{%- capture actualWidths -%}
			{{ actualWidths }},
			{%- render 'img--width',
				img: img,
				width: widthNum,
				height: heightNum
			-%}
		{%- endcapture -%}
		{%- if img.width <= widthNum or img.height <= heightNum -%}
			{% assign hasMaxedOut = true %}
		{%- endif -%}
	{%- endunless -%}
{%- endfor -%}
{%- assign actualWidthsLength = actualWidths | size | minus: 1 -%}
{%- assign actualWidthsTrimmed = actualWidths | slice: 0, actualWidthsLength -%}
{{ actualWidthsTrimmed | join: ',' }}
```
