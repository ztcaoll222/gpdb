# src/pl/plpython/Makefile

subdir = src/pl/plpython
top_builddir = ../../..
include $(top_builddir)/src/Makefile.global


# On some platforms we can only build PL/Python if libpython is a
# shared library.  Since there is no official way to determine this
# (at least not in pre-2.3 Python), we see if there is a file that is
# named like a shared library.
ifneq (,$(wildcard $(python_libdir)/libpython*$(DLSUFFIX)*))
shared_libpython = yes
endif

# Windows needs to convert backslashed paths to normal slashes,
# and we have to remove -lpython from the link since we are building our own
ifeq ($(PORTNAME), win32)
shared_libpython = yes
python_includespec := $(subst \,/,$(python_includespec))
override python_libspec =
endif

# Darwin (OS X) has its own ideas about how to do this.
ifeq ($(PORTNAME), darwin)
shared_libpython = yes
# Replaced to allow use of Greenplum-provided Python
#override python_libspec = -framework Python
#override python_additional_libs =
# sys.prefix reflects the runtime location of the python tree;
# the GP install has the dylib in the lib directory of that tree.
override python_libspec = -L$(shell ${PYTHON} -c "import sys, os; print os.path.join(sys.prefix, 'lib')") -lpython${python_version}
endif

override python_libspec := -L${python_configdir}/../.. ${python_libspec}

# If we don't have a shared library and the platform doesn't allow it
# to work without, we have to skip it.
ifneq (,$(findstring yes, $(shared_libpython)$(allow_nonpic_in_shlib)))

# do not fail build due to warnings in this code
#override CPPFLAGS := -Wno-error -I. -I$(srcdir) $(python_includespec) $(CPPFLAGS) -DPLPYTHON_SHOW_DEBUG_INFO
override CPPFLAGS := -Wno-error -I. -I$(srcdir) $(python_includespec) $(CPPFLAGS)


rpathdir = $(python_libdir):$(INSTLOC)/ext/python/lib

NAME = plpython$(python_majorversion)

OBJS = plpython.o

DATA = plpythonu.control plpythonu--1.0.sql plpythonu--unpackaged--1.0.sql \
       plpython2u.control plpython2u--1.0.sql plpython2u--unpackaged--1.0.sql \
       plpython3u.control plpython3u--1.0.sql plpython3u--unpackaged--1.0.sql


# Python on win32 ships with import libraries only for Microsoft Visual C++,
# which are not compatible with mingw gcc. Therefore we need to build a
# new import library to link with.
ifeq ($(PORTNAME), win32)
pytverstr=$(subst .,,${python_version})
OBJS += libpython${pytverstr}.a
libpython${pytverstr}.a: python${pytverstr}.def
	dlltool --dllname python${pytverstr}.dll --def python${pytverstr}.def --output-lib  libpython${pytverstr}.a
WD=$(subst \,/,$(WINDIR))
python${pytverstr}.def: $(WD)/system32/python${pytverstr}.dll
	pexports $(WD)/system32/python${pytverstr}.dll > python${pytverstr}.def
endif


SHLIB_LINK = $(python_libspec) $(python_additional_libs) $(filter -lintl,$(LIBS))

REGRESS_OPTS = --dbname=$(PL_TESTDB) --init-file=$(srcdir)/init_file
# Only load plpythonu with Python 2.  The test files themselves load
# the versioned language plpython(2|3)u.
ifeq ($(python_majorversion),2)
REGRESS_OPTS += --load-extension=plpythonu
endif

REGRESS = \
	plpython_schema \
	plpython_populate \
	plpython_test \
	plpython_do \
	plpython_global \
	plpython_import \
	plpython_spi \
	plpython_newline \
	plpython_void \
	plpython_params \
	plpython_setof \
	plpython_record \
	plpython_trigger \
	plpython_types \
	plpython_error \
	plpython_unicode \
	plpython_quote \
	plpython_composite \
	plpython_subtransaction \
	plpython_returns \
	plpython_gpdb \
	plpython_drop

# where to find psql for running the tests
PSQLDIR = $(bindir)

include $(top_srcdir)/src/Makefile.shlib

# Force this dependency to be known even without dependency info built:
plpython.o: spiexceptions.h

spiexceptions.h: $(top_srcdir)/src/backend/utils/errcodes.txt generate-spiexceptions.pl
	$(PERL) $(srcdir)/generate-spiexceptions.pl $< > $@

all: all-lib

distprep: spiexceptions.h


install: all install-lib install-data

installdirs: installdirs-lib
	$(MKDIR_P) '$(DESTDIR)$(datadir)/extension'

uninstall: uninstall-lib uninstall-data

install-data: installdirs
	$(INSTALL_DATA) $(addprefix $(srcdir)/, $(DATA)) '$(DESTDIR)$(datadir)/extension/'

uninstall-data:
	rm -f $(addprefix '$(DESTDIR)$(datadir)/extension'/, $(notdir $(DATA)))

.PHONY: install-data uninstall-data


ifeq ($(python_majorversion),3)
# Adjust regression tests for Python 3 compatibility
prep3:
	$(MKDIR_P) python3 python3/sql python3/expected
	for file in $(srcdir)/sql/*.sql $(srcdir)/expected/*.out; do \
	  sed -e 's/except \([[:alpha:]][[:alpha:].]*\), *\([[:alpha:]][[:alpha:]]*\):/except \1 as \2:/g' \
	      -e "s/<type 'exceptions\.\([[:alpha:]]*\)'>/<class '\1'>/g" \
	      -e "s/<type 'long'>/<class 'int'>/g" \
	      -e "s/\([0-9][0-9]*\)L/\1/g" \
	      -e 's/\([ [{]\)u"/\1"/g' \
	      -e "s/\([ [{]\)u'/\1'/g" \
	      -e "s/def next/def __next__/g" \
	      -e "s/LANGUAGE plpythonu/LANGUAGE plpython3u/g" \
	      -e "s/LANGUAGE plpython2u/LANGUAGE plpython3u/g" \
	      -e "s/EXTENSION plpythonu/EXTENSION plpython3u/g" \
	      -e "s/EXTENSION plpython2u/EXTENSION plpython3u/g" \
	    $$file >`echo $$file | sed 's,$(srcdir),python3,'` || exit; \
	done

clean3:
	rm -rf python3/

check: submake prep3
	$(pg_regress_check) --inputdir=./python3 --outputdir=./python3 $(REGRESS_OPTS) $(REGRESS)

installcheck: submake prep3
	$(pg_regress_installcheck) --inputdir=./python3 --outputdir=./python3 $(REGRESS_OPTS) $(REGRESS)

clean: clean3
else
check: submake
	$(pg_regress_check) $(REGRESS_OPTS) $(REGRESS)

installcheck: submake
	$(pg_regress_installcheck) $(REGRESS_OPTS) $(REGRESS)
endif

.PHONY: submake
submake:
	$(MAKE) -C $(top_builddir)/src/test/regress pg_regress$(X)

clean distclean: clean-lib
	rm -f $(OBJS)
	rm -rf $(pg_regress_clean_files)
ifeq ($(PORTNAME), win32)
	rm -f python${pytverstr}.def
endif

maintainer-clean: distclean
	rm -f spiexceptions.h

else # can't build

all:
	@echo ""; \
	 echo "*** Cannot build PL/Python because libpython is not a shared library." ; \
	 echo "*** You might have to rebuild your Python installation.  Refer to"; \
	 echo "*** the documentation for details."; \
	 echo ""

endif # can't build
