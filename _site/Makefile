.PHONY: new
new:
	docker run --rm --label=jekyll --volume=$(shell pwd):/srv/jekyll -it -p 4000:4000 jekyll/jekyll jekyll new .

.PHONY: serve
serve:
	docker run --rm --label=jekyll --volume=$(shell pwd):/srv/jekyll -it -p 4000:4000 jekyll/jekyll

.DEFAULT_GOAL := serve
