project(
  'nanobind',
  'cpp',
  # 'src/version.py' prints the nanobind and nanobind-backend versions on
  # separate lines; the project version is the first one.
  version: run_command(
    'src/version.py',
    check: true,
  ).stdout().strip().split('\n')[0].strip(),
  meson_version: '>=1.0.0',
  default_options: [
    'cpp_std=c++17',
    'b_ndebug=if-release',
    'default_library=static',
  ],
  license: 'BSD-3-Clause',
)

py_mod = import('python')
fs = import('fs')
py = py_mod.find_installation()
py_dep = py.dependency()

# Prefer the bundled robin-map headers (like the CMake build's git submodule).
# This also avoids configuring the wrap subproject and building its test suite.
if fs.is_dir(meson.current_source_dir() / 'ext/robin_map/include')
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

# Compile/link arguments, grouped to mirror the CMake configuration:
#   'core_*' <- nanobind_build_library()  (the compiled nanobind core)
#   'ext_*'  <- nanobind_add_module()      (a binding extension)
#   'pub_*'  <- flags CMake marks PUBLIC on libnanobind (reach both)
# The 'nanobind' dependency compiles the core into the consumer's extension, so
# it transports the union of these groups.

compiler = meson.get_compiler('cpp')
is_msvc = compiler.get_argument_syntax() == 'msvc'
is_windows = host_machine.system() == 'windows'
is_darwin = host_machine.system() == 'darwin'

# NB_OPT / NB_OPT_SIZE generator expressions mapped to meson build types:
#   NB_OPT_SIZE = Release, MinSizeRel, RelWithDebInfo  (any optimized build)
#   NB_OPT      = Release, MinSizeRel                  (optimized, no debug info)
nb_opt_size = get_option('optimization') in ['1', '2', '3', 's']
nb_opt = nb_opt_size and not get_option('debug')

freethreaded = py.get_variable('Py_GIL_DISABLED', 0) != 0

# Probe for the faster TLSDESC thread-local model (Linux/x86_64)
nb_has_mtls_gnu2 = false
if not (is_msvc or is_windows or is_darwin)
  nb_has_mtls_gnu2 = compiler.has_argument('-mtls-dialect=gnu2')
endif

# PUBLIC flags on libnanobind: hidden visibility, plus section splitting so the
# linker can drop unused core code in optimized builds.
pub_args = []
pub_link = []
if not is_msvc
  pub_args += '-fvisibility=hidden'
endif
if not is_windows and not is_darwin and nb_opt_size
  pub_args += ['-ffunction-sections', '-fdata-sections']
  pub_link += '-Wl,--gc-sections'
endif

# nanobind_build_library(): core-only compile flags
core_only_args = []
if is_msvc
  core_only_args += '/D_CRT_SECURE_NO_WARNINGS'
else
  core_only_args += '-fno-strict-aliasing'
endif
if freethreaded
  core_only_args += '-DNB_FREE_THREADED'
endif
if nb_has_mtls_gnu2
  core_only_args += '-mtls-dialect=gnu2'
endif

# nanobind_add_module(): extension-only compile flags
ext_only_args = []
if is_msvc
  ext_only_args += '/bigobj'                            # nanobind_compile_options
  if compiler.get_id() == 'msvc'
    ext_only_args += '/MP'
  endif
endif
if nb_opt_size
  ext_only_args += is_msvc ? '/Os' : '-Os'             # nanobind_opt_size
endif
if nb_opt
  ext_only_args += is_msvc ? '/GS-' : '-fno-stack-protector'  # nanobind_disable_stack_protector
endif

# nanobind_strip() + nanobind_link_options() (macOS linker response file)
strip_link = []
if nb_opt
  if is_darwin
    strip_link += ['-Wl,-dead_strip', '-Wl,-x', '-Wl,-S']
  elif not is_windows
    strip_link += '-Wl,-s'
  endif
endif
if is_darwin
  py_soabi = py.get_variable('SOABI')
  if py_soabi.startswith('cp')
    resp_file = meson.project_source_root() / 'cmake/darwin-ld-cpython.sym'
  else
    resp_file = meson.project_source_root() / 'cmake/darwin-ld-pypy.sym'
  endif
  strip_link += '-Wl,@' + resp_file
endif

# NB_COMPACT_ASSERTIONS: the linked core in optimized builds. The split backend
# keeps full assertions (FULL_ASSERTIONS) and therefore omits it.
compact_assert = nb_opt_size ? ['-DNB_COMPACT_ASSERTIONS'] : []

# Per-target groups consumed by the test suite (tests/meson.build)
core_compile_args = pub_args + core_only_args + compact_assert
core_backend_args = pub_args + core_only_args
ext_compile_args = pub_args + ext_only_args
ext_link_args = strip_link + pub_link

# Shared-library build: export the core (NB_SHARED/NB_BUILD select NB_CORE)
shared_defs = []
if get_option('default_library') == 'shared'
  shared_defs += ['-DNB_SHARED', '-DNB_BUILD']
endif

# The dependency compiles the core into the consumer's extension, so it carries
# the union of the core and extension groups.
dep_compile_args = core_compile_args + ext_only_args + shared_defs
dep_link_args = ext_link_args

nanobind_sources = files('src/nb_combined.cpp')

# Includes-only Python dependency (no link arguments). Stable-ABI builds use
# this in place of 'py_dep' so that nanobind does not drag in the version-tagged
# 'pythonXY' import library, which would otherwise defeat the limited API on
# Windows (meson issue #15172).
py_dep_inc = py_dep.partial_dependency(compile_args: true, includes: true)

# On Windows, a stable-ABI extension links against 'python3.lib' rather than
# 'pythonXY.lib'. Recent meson can fail to add it (issue #15479), so locate it
# next to the version-tagged import library and link it explicitly.
py_limited_dep = declare_dependency()
if host_machine.system() == 'windows'
  py_libdir = run_command(
    py,
    '-c',
    'import sysconfig, os; print(os.path.join(sysconfig.get_config_var("installed_base"), "libs"))',
    check: true,
  ).stdout().strip()
  py_limited_dep = compiler.find_library('python3', dirs: [py_libdir])
endif

stubgen_path = meson.current_source_dir() / 'src/stubgen.py'

stable_abi = get_option('stable_abi')

if stable_abi != ''
  # Linked mode against the stable ABI: compile the core sources, but link the
  # stable 'python3' import library on Windows (see py_limited_dep above).
  nanobind_dep = declare_dependency(
    sources: nanobind_sources,
    include_directories: [incdir],
    dependencies: [py_dep_inc, py_limited_dep, robin_map_dep],
    compile_args: dep_compile_args,
    link_args: dep_link_args,
    variables: {'stubgen': stubgen_path},
  )
else
  nanobind_dep = declare_dependency(
    sources: nanobind_sources,
    include_directories: [incdir],
    dependencies: [py_dep, robin_map_dep],
    compile_args: dep_compile_args,
    link_args: dep_link_args,
    variables: {'stubgen': stubgen_path},
  )
endif

meson.override_dependency('nanobind', nanobind_dep)
