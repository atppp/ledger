# Build PDF or HTML from the markdown notes in the repo root.
#
# From the repo root:  make -C build
#                      make -C build pdf
# From this directory: make
#                      make pdf
#
# PDF prerequisites (Homebrew):
#   brew install pandoc
#   brew install --cask mactex-no-gui   # or full mactex; provides xelatex
#   # After MacTeX install, ensure /Library/TeX/texbin is on PATH
#   # (new shells usually pick it up via /etc/paths.d).
# Code-block boxes need the tcolorbox LaTeX package (included with MacTeX;
# if missing: tlmgr install tcolorbox).
#
# HTML needs only pandoc. Math is rendered in the browser via MathJax.
# Styles and the mobile TOC menu live in head.html (inlined into <head>), so
# ledger.html is self-contained aside from MathJax's CDN.

ROOT := ..

CHAPTERS := \
	$(ROOT)/intro.md \
	$(ROOT)/beancount.md \
	$(ROOT)/importing-bank.md \
	$(ROOT)/importing-market.md \
	$(ROOT)/importing-private.md \
	$(ROOT)/performance.md

HEADER := pdf-header.tex
HTML_HEAD := head.html
PDF := ledger.pdf
HTML := ledger.html
PDF_ENGINE ?= xelatex

.PHONY: html pdf clean

html: $(HTML)

pdf: $(PDF)

$(PDF): $(CHAPTERS) $(HEADER)
	pandoc $(CHAPTERS) \
		-o $@ \
		--pdf-engine=$(PDF_ENGINE) \
		-H $(HEADER) \
		-V geometry:margin=1in \
		--toc \
		--toc-depth=2 \
		--syntax-highlighting=tango

$(HTML): $(CHAPTERS) $(HTML_HEAD) Makefile
	pandoc $(CHAPTERS) \
		-o $@ \
		-s \
		-V pagetitle="Building a Beancount Ledger" \
		-H $(HTML_HEAD) \
		--toc \
		--toc-depth=2 \
		--math-method=mathjax \
		--syntax-highlighting=tango

clean:
	rm -f $(PDF) $(HTML)
