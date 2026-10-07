project(
  'nanobind',
  'cpp',
  version: run_command(
    'src/version.py',
    check: true,
  ).stdout().strip(),
  meson_version: '>=1.1.0',
  default_options: [
    'cpp_std=c++17',
    'b_ndebug=if-release',
    'default_library=static',
  ],
  license: 'BSD-3-Clause',
)

py_mod = import('python')
py = py_mod.find_installation()
py_dep = py.dependency()

# Prefer the bundled robin_map submodule (ext/robin_map), matching the upstream
# CMake build. This keeps the build self-contained (no download / system
# dependency needed, which matters for the backend conda package). Fall back to
# a system / wrap 'tsl-robin-map' when the submodule is not checked out.
fs = import('fs')
if fs.is_dir(meson.current_source_dir() / 'ext' / 'robin_map' / 'include')
  robin_map_dep = declare_dependency(
    include_directories: include_directories('ext/robin_map/include'),
  )
else
  robin_map_dep = dependency(
    'tsl-robin-map',
    version: '>=1.4.0',
  )
endif

incdir = include_directories('include')

# Arguments are derived from nanobind_build_library() in CMake configuration.
dep_compile_args = []
dep_link_args = []

compiler = meson.get_compiler('cpp')
if compiler.get_argument_syntax() == 'msvc'
  dep_compile_args += ['/D_CRT_SECURE_NO_WARNINGS']
endif

# Discussion in WrapDB PR #1556 as to whether or not the following is necessary:
# dep_compile_args += ['-fno-strict-aliasing']

freethreaded = py.get_variable('Py_GIL_DISABLED', 0) != 0
if freethreaded
  dep_compile_args += ['-DNB_FREE_THREADED']
endif

if not get_option('debug')
  dep_compile_args += ['-DNB_COMPACT_ASSERTIONS']

  # The following, from nanobind_strip(), applies both to the nanobind shared
  # library (if it's built) and to the developer's extension:
  if host_machine.system() == 'darwin'
    dep_link_args += ['-Wl,-dead_strip', '-Wl,-x', '-Wl,-S']
  elif host_machine.system() != 'windows'
    dep_link_args += ['-Wl,-s']
  endif
endif

if host_machine.system() == 'darwin'
  py_soabi = py.get_variable('SOABI')
  if py_soabi.startswith('cp')
    resp_file = meson.project_source_root() / 'cmake/darwin-ld-cpython.sym'
  else
    resp_file = meson.project_source_root() / 'cmake/darwin-ld-pypy.sym'
  endif
  dep_link_args += ['-Wl,@' + resp_file]
endif

if get_option('default_library') == 'shared'
  dep_compile_args += ['-DNB_SHARED']
  add_project_arguments(
    [dep_compile_args, '-DNB_BUILD'],
    language: 'cpp',
  )
  add_project_link_arguments(
    dep_link_args,
    language: 'cpp',
  )
elif host_machine.system() != 'windows' and host_machine.system() != 'darwin'
  dep_compile_args += ['-ffunction-sections', '-fdata-sections']
  add_project_arguments(
    dep_compile_args,
    language: 'cpp',
  )
  dep_link_args += ['-Wl,--gc-sections']
else
  add_project_arguments(
    dep_compile_args,
    language: 'cpp',
  )
endif

nanobind_sources = files('src/nb_combined.cpp')

# -- Classic (linked) dependency ------------------------------------------
# Combined frontend + backend: compiles libnanobind (nb_combined.cpp) straight
# into every extension. This is the default, unchanged behaviour.
nanobind_dep = declare_dependency(
  sources: nanobind_sources,
  include_directories: [incdir],
  dependencies: [py_dep, robin_map_dep],
  compile_args: dep_compile_args,
  link_args: dep_link_args,
  variables: {
    'stubgen': meson.current_source_dir() / 'src/stubgen.py',
  },
)

meson.override_dependency('nanobind', nanobind_dep)

# -- Split mode (nanobind >= 3.0) -----------------------------------------
# The frontend (binding code) and the backend (libnanobind) live in separate
# Python modules. See https://nanobind.readthedocs.io/en/latest/split_mode.html
#
# 'nanobind-frontend' is a HEADER-ONLY dependency: it does NOT compile
# nb_combined.cpp. Instead each extension imports the shared backend module at
# runtime (NB_BACKEND_MODULE). The consumer must additionally build its
# extension with 'limited_api: nanobind_stable_abi' so the module targets the
# Python stable ABI (abi3) that split mode requires. The result is a single
# abi3 wheel per platform that works on every Python >= the floor, and multiple
# independently built extensions reliably share C++ types through the common
# backend.
nanobind_backend_module = get_option('nb_backend_module')
nanobind_backend_pypi = get_option('nb_backend_pypi')
nanobind_stable_abi = get_option('nb_stable_abi')

nanobind_frontend_dep = declare_dependency(
  include_directories: [incdir],
  # Headers only: provide the Python *include dirs* (partial_dependency) but NOT
  # the Python link library. In split mode every consumer is an abi3
  # extension_module whose own (limited_api) Python dependency links the stable
  # python3.lib. Linking the full version-specific python3XX.lib here as well
  # would make the resulting module depend on python3XX.dll, breaking the
  # one-abi3-module-per-platform promise on Windows (it would only load on the
  # build's Python version). See the limited_api handling in Meson's python
  # module.
  dependencies: [
    py_dep.partial_dependency(compile_args: true, includes: true),
    robin_map_dep,
  ],
  compile_args: dep_compile_args + [
    '-DNB_BACKEND_MODULE=' + nanobind_backend_module,
    '-DNB_BACKEND_PYPI=' + nanobind_backend_pypi,
  ],
  link_args: dep_link_args,
  variables: {
    'stubgen': meson.current_source_dir() / 'src/stubgen.py',
    'stable_abi': nanobind_stable_abi,
    'backend_module': nanobind_backend_module,
    'backend_pypi': nanobind_backend_pypi,
  },
)

meson.override_dependency('nanobind-frontend', nanobind_frontend_dep)

# -- Backend module --------------------------------------------------------
# Build and install the compiled backend module served to split-mode
# extensions (equivalent to the CMake nanobind_add_backend() command). This is
# how the 'nanobind-backend' conda/pip package is produced.
#
# Performance: the backend contains the perf-critical dispatcher, so it is
# compiled at the project optimization level (-O3 in a release build) and with
# FULL assertions kept (NB_COMPACT_ASSERTIONS is intentionally NOT defined,
# because the backend ships prebuilt and its users cannot rebuild it in debug
# mode to recover a detailed message). Unused code is still trimmed via
# function/data sections + gc-sections so the module stays small.
if get_option('nb_build_backend')
  backend_compile_args = [
    '-DNB_BUILD',
    '-DNB_BACKEND_NAME=' + nanobind_backend_module,
  ]
  backend_link_args = []

  if freethreaded
    backend_compile_args += ['-DNB_FREE_THREADED']
  endif

  if compiler.get_argument_syntax() == 'msvc'
    backend_compile_args += ['/D_CRT_SECURE_NO_WARNINGS', '/bigobj']
  else
    # type punning in the raw CPython API code (see nb_combined.cpp notes)
    backend_compile_args += ['-fno-strict-aliasing']

    if not get_option('debug')
      # trim unused parts of libnanobind out of the module
      backend_compile_args += ['-ffunction-sections', '-fdata-sections']
      if host_machine.system() == 'darwin'
        backend_link_args += ['-Wl,-dead_strip', '-Wl,-x', '-Wl,-S']
      else
        backend_link_args += ['-Wl,-s', '-Wl,--gc-sections']
      endif
    endif

    # Faster GNU2 (TLS descriptor) thread-local model, matching upstream CMake.
    # Off by default here for portability (see meson_options.txt).
    if get_option('nb_tls_dialect_gnu2') and compiler.has_argument('-mtls-dialect=gnu2')
      backend_compile_args += ['-mtls-dialect=gnu2']
    endif
  endif

  if host_machine.system() == 'darwin'
    py_soabi = py.get_variable('SOABI')
    if py_soabi.startswith('cp')
      backend_resp_file = meson.project_source_root() / 'cmake/darwin-ld-cpython.sym'
    else
      backend_resp_file = meson.project_source_root() / 'cmake/darwin-ld-pypy.sym'
    endif
    backend_link_args += ['-Wl,@' + backend_resp_file]
  endif

  # The backend is a normal (full-API, version-specific) extension module -- it
  # is NOT limited-API. nb_backend.cpp provides the module entry point and the
  # fill() method; nb_combined.cpp provides the libnanobind implementation.
  py.extension_module(
    nanobind_backend_module,
    ['src/nb_backend.cpp', 'src/nb_combined.cpp'],
    include_directories: [incdir],
    dependencies: [py_dep, robin_map_dep],
    cpp_args: backend_compile_args,
    link_args: backend_link_args,
    gnu_symbol_visibility: 'hidden',
    install: true,
    subdir: '',
  )

  message('nanobind backend module enabled: ' + nanobind_backend_module)
endif
