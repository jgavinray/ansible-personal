SHELL := bash

MKDOCSIMAGE = docker.io/squidfunk/mkdocs-material:5.2.3

help: ## Print this help
	@(export SEP=$$'\01'; grep -E "^[a-zA-Z0-9_-]+:.*?## .*$$" $(MAKEFILE_LIST) | sed "s/:.*##/$$SEP/" | column -ts "$$SEP" -c 120)

documentation: ## Generate the documentation site under $OUTPUT_DIR (default: ./docs)
	@docker run --rm -it -p 8000:8000 -v ${PWD}:/docs ${MKDOCSIMAGE}
