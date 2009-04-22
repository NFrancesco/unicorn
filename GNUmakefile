# use GNU Make to run tests in parallel, and without depending on Rubygems
all:: test
ruby = ruby
ragel = ragel
RLFLAGS = -G2
-include local.mk
ruby_bin := $(shell which $(ruby))
ifeq ($(DLEXT),) # "so" for Linux
  DLEXT := $(shell $(ruby) -rrbconfig -e 'puts Config::CONFIG["DLEXT"]')
endif
ifeq ($(RUBY_VERSION),)
  RUBY_VERSION := $(shell $(ruby) -e 'puts RUBY_VERSION')
endif

# dunno how to implement this as concisely in Ruby, and hell, I love awk
awk_slow := awk '/def test_/{print FILENAME"--"$$2".n"}' 2>/dev/null

rails_vers := $(subst test/rails/app-,,$(wildcard test/rails/app-*))
slow_tests := test/unit/test_server.rb test/exec/test_exec.rb \
  test/unit/test_signals.rb
log_suffix = .$(RUBY_VERSION).log
T_r := $(wildcard test/rails/test*.rb)
T := $(filter-out $(slow_tests) $(T_r), $(wildcard test/*/test*.rb))
T_n := $(shell $(awk_slow) $(slow_tests))
T_log := $(subst .rb,$(log_suffix),$(T))
T_n_log := $(subst .n,$(log_suffix),$(T_n))
T_r_log := $(subst .r,$(log_suffix),$(T_r))
test_prefix = $(CURDIR)/test/install-$(RUBY_VERSION)

ext := ext/unicorn/http11
c_files := $(addprefix $(ext)/,ext_help.h http11.c http11_parser.h)
rl_files := $(addprefix $(ext)/,http11_parser.rl http11_parser_common.rl)
rb_files := $(shell grep '^\(bin\|lib\)' Manifest)
inst_deps := $(c_files) $(rb_files)

ragel: $(ext)/http11_parser.h
$(ext)/http11_parser.h: $(rl_files)
	cd $(@D) && $(ragel) http11_parser.rl -C $(RLFLAGS) -o $(@F)
	$(ruby) -i -p -e '$$_.gsub!(%r{[ \t]*$$},"")' $@
$(ext)/Makefile: $(ext)/extconf.rb $(c_files)
	cd $(@D) && $(ruby) extconf.rb
$(ext)/http11.$(DLEXT): $(ext)/Makefile
	$(MAKE) -C $(@D)
lib/unicorn/http11.$(DLEXT): $(ext)/http11.$(DLEXT)
	@mkdir -p lib
	install -m644 $< $@
http11: lib/unicorn/http11.$(DLEXT)

$(test_prefix)/.stamp: $(inst_deps)
	mkdir -p $(test_prefix)/.ccache
	tar c bin ext lib GNUmakefile Manifest | (cd $(test_prefix) && tar x)
	$(MAKE) -C $(test_prefix) clean
	$(MAKE) -C $(test_prefix) http11 shebang
	> $@

# this is only intended to be run within $(test_prefix)
shebang: bin/unicorn bin/unicorn_rails
	$(ruby) -i -p -e '$$_.gsub!(%r{^#!.*$$},"#!$(ruby_bin)")' $^

t_log := $(T_log) $(T_n_log)
test: $(T) $(T_n)
	@cat $(t_log) | $(ruby) test/aggregate.rb
	@$(RM) $(t_log)

test-exec: $(wildcard test/exec/test_*.rb)
test-unit: $(wildcard test/unit/test_*.rb)
$(slow_tests): $(test_prefix)/.stamp
	@$(MAKE) $(shell $(awk_slow) $@)

TEST_OPTS = -v
TEST_OPTS = -v
ifndef V
       quiet_pre = @echo '* $(arg)$(extra)';
       quiet_post = >$(t) 2>&1
else
       # we can't rely on -o pipefail outside of bash 3+,
       # so we use a stamp file to indicate success and
       # have rm fail if the stamp didn't get created
       stamp = $@$(log_suffix).ok
       quiet_pre = @echo $(ruby) $(arg) $(TEST_OPTS); ! test -f $(stamp) && (
       quiet_post = && > $(stamp) )>&2 | tee $(t); rm $(stamp) 2>/dev/null
endif
run_test = $(quiet_pre) setsid $(ruby) -w $(arg) $(TEST_OPTS) $(quiet_post) || \
  (sed "s,^,$(extra): ," >&2 < $(t); exit 1)

%.n: arg = $(subst .n,,$(subst --, -n ,$@))
%.n: t = $(subst .n,$(log_suffix),$@)
%.n: export PATH := $(test_prefix)/bin:$(PATH)
%.n: export RUBYLIB := $(test_prefix)/lib:$(RUBYLIB)
%.n: $(test_prefix)/.stamp
	$(run_test)

$(T): arg = $@
$(T): t = $(subst .rb,$(log_suffix),$@)
$(T): export PATH := $(test_prefix)/bin:$(PATH)
$(T): export RUBYLIB := $(test_prefix)/lib:$(RUBYLIB)
$(T): $(test_prefix)/.stamp
	$(run_test)

install: bin/unicorn bin/unicorn_rails
	$(prep_setup_rb)
	$(RM) -r .install-tmp
	mkdir .install-tmp
	cp -p $^ .install-tmp
	$(ruby) setup.rb all
	$(RM) $^
	mv $(addprefix .install-tmp/,$(^F)) bin/
	$(RM) -r .install-tmp
	$(prep_setup_rb)

setup_rb_files := .config InstalledFiles
prep_setup_rb := @-$(RM) $(setup_rb_files);$(MAKE) -C $(ext) clean

clean:
	-$(MAKE) -C $(ext) clean
	$(RM) $(ext)/Makefile lib/unicorn/http11.$(DLEXT)
	$(RM) $(setup_rb_files) $(t_log)
	$(RM) -r $(test_prefix)

Manifest:
	git ls-files > $@+
	cmp $@+ $@ || mv $@+ $@
	$(RM) -f $@+

# using rdoc 2.4.1
doc: .document
	rdoc -Na -m README -t "$(shell sed -ne '1s/^= //p' README)"

rails_git_url = git://github.com/rails/rails.git
rails_git := vendor/rails.git
$(rails_git)/info/cloned-stamp:
	git clone --mirror -q $(rails_git_url) $(rails_git)
	> $@

rails_tests := $(addsuffix .r,$(addprefix $(T_r).,$(rails_vers)))
test-rails: $(rails_tests)
$(T_r).%.r: t = $(addsuffix $(log_suffix),$@)
$(T_r).%.r: rv = $(subst .r,,$(subst $(T_r).,,$@))
$(T_r).%.r: extra = ' 'v$(rv)
$(T_r).%.r: arg = $(T_r)
$(T_r).%.r: export PATH := $(test_prefix)/bin:$(PATH)
$(T_r).%.r: export RUBYLIB := $(test_prefix)/lib:$(RUBYLIB)
$(T_r).%.r: export UNICORN_RAILS_TEST_VERSION = $(rv)
$(T_r).%.r: export RAILS_GIT_REPO = $(CURDIR)/$(rails_git)
$(T_r).%.r: $(test_prefix)/.stamp $(rails_git)/info/cloned-stamp
	$(run_test)

.PHONY: doc $(T) $(slow_tests) Manifest
