# Port of tests/CMakeLists.txt to Meson. Built when '-Dbuild_tests=true'. The
# nanobind core is compiled once into a static library (linked mode) or served
# by a local backend module (split mode); every test extension links or
# delegates to it. Test inputs are copied next to the built extensions so that
# 'pytest' can run against them in the build directory.

fs = import('fs')

# Abort during interpreter shutdown when leaks are found, to catch this in CI.
# nb_internals.cpp reads this, so it must reach every test target (mirrors the
# CMake add_definitions()).
test_defs = ['-DNB_ABORT_ON_LEAK']
add_project_arguments(test_defs, language: 'cpp')

split_mode = get_option('test_split_mode')
stable_abi_test = get_option('test_stable_abi')
backend_name = 'nanobind_backend_test'

# Every extension and stub is collected here and declared as a dependency of the
# 'pytest' test below, so that a bare 'meson test' builds them first.
test_deps = []

# Extension-flavor selection. 'ext_kwargs' is expanded into every test
# extension; 'core_extra' adjusts the shared core; 'frontend_defs' carries the
# split-mode backend selector.
ext_kwargs = {
  'include_directories': incdir,
  'link_args': ext_link_args,
  'install': false,
}
core_extra = []
frontend_defs = []

if split_mode
  abi_floor = '3.10'
  ext_kwargs += {'limited_api': abi_floor}
  frontend_defs += '-DNB_BACKEND_MODULE=' + backend_name
elif stable_abi_test
  abi_floor = '3.12'
  parts = abi_floor.split('.')
  hexval = parts[0].to_int() * 16777216 + parts[1].to_int() * 65536
  # The core library is not built via extension_module(), so it needs the
  # Py_LIMITED_API define spelled out (extension_module(limited_api:) adds it to
  # the frontend translation units and picks the abi3 suffix).
  core_extra = ['-DPy_LIMITED_API=' + hexval.to_string()]
  ext_kwargs += {'limited_api': abi_floor}
endif

# The shared nanobind core (linked mode) or the backend module (split mode).
base_link = []
if split_mode
  # nanobind_add_backend(): full assertions (no NB_COMPACT_ASSERTIONS) plus the
  # extension-level size/strip options.
  backend_lib = py.extension_module(
    backend_name,
    [nanobind_sources, nanobind_backend_sources],
    include_directories: incdir,
    dependencies: [py_dep, robin_map_dep],
    cpp_args: core_backend_args + ext_only_args
      + ['-DNB_BUILD', '-DNB_BACKEND_NAME=' + backend_name],
    link_args: ext_link_args,
    install: false,
  )
  stub_deps = [backend_lib]
  test_deps += backend_lib
else
  core_lib = static_library(
    'nanobind_test_core',
    nanobind_sources,
    include_directories: incdir,
    dependencies: [py_dep_inc, robin_map_dep],
    cpp_args: core_compile_args + core_extra,
  )
  base_link = [core_lib]
  stub_deps = []
endif

# ---------------------------------------------------------------------------
# Test extensions
# ---------------------------------------------------------------------------

extra_sources = {
  'classes': ['test_classes_extra.cpp'],
  'functions': ['test_abi_layout.cpp'],
  'intrusive': ['test_intrusive_impl.cpp'],
}

test_names = [
  'accessor', 'functions', 'callbacks', 'classes', 'holders', 'stl',
  'stl_bind_map', 'stl_bind_vector', 'chrono', 'enum', 'eval', 'ndarray',
  'jax', 'tensorflow', 'exception', 'make_iterator', 'typing', 'issue',
  'intrusive', 'thread',
]

test_ext = {}
foreach name : test_names
  srcs = ['test_' + name + '.cpp']
  if name in extra_sources
    srcs += extra_sources[name]
  endif
  mod = py.extension_module(
    'test_' + name + '_ext',
    srcs,
    cpp_args: ext_compile_args + frontend_defs,
    link_with: base_link,
    kwargs: ext_kwargs,
  )
  test_ext += {name: mod}
  test_deps += mod
endforeach

# Bindings that exchange types through a shared library (own type domain)
inter_module_lib = shared_library(
  'inter_module',
  'inter_module.cpp',
  include_directories: incdir,
  dependencies: py_dep_inc,
  cpp_args: ['-DSHARED_BUILD'],
  install: false,
)

domain_defs = frontend_defs + ['-DNB_DOMAIN=mydomain']
foreach n : ['1', '2']
  test_deps += py.extension_module(
    'test_inter_module_' + n + '_ext',
    'test_inter_module_' + n + '.cpp',
    cpp_args: ext_compile_args + domain_defs,
    link_with: base_link + [inter_module_lib],
    kwargs: ext_kwargs,
  )
endforeach

# An extension that sets up nanobind via nb::register_module()
test_deps += py.extension_module(
  'test_foreign_ext',
  'test_foreign.cpp',
  cpp_args: ext_compile_args + frontend_defs,
  link_with: base_link,
  kwargs: ext_kwargs,
)

# Eigen bindings, only when Eigen is available
eigen_dep = dependency('eigen3', required: false)
if eigen_dep.found()
  foreach name : ['eigen', 'eigen_tensor']
    test_deps += py.extension_module(
      'test_' + name + '_ext',
      'test_' + name + '.cpp',
      cpp_args: ext_compile_args + frontend_defs,
      dependencies: eigen_dep,
      link_with: base_link,
      kwargs: ext_kwargs,
    )
  endforeach
endif

# Split-mode-only extensions
if split_mode
  abi_multi_srcs = ['test_abi_multi_1.cpp', 'test_abi_multi_2.cpp']

  # Bindings spread over two translation units that exchange bound types; the
  # LTO twin re-checks this under interprocedural optimization.
  test_deps += py.extension_module(
    'test_abi_multi_ext',
    abi_multi_srcs,
    cpp_args: ext_compile_args + frontend_defs,
    kwargs: ext_kwargs,
  )

  # The LTO twin needs a toolchain that can link with -flto (e.g. the LLVM gold
  # plugin). Skip it otherwise; test_abi.py skips the matching parameter.
  if compiler.links('int main() { return 0; }', args: ['-flto'], name: 'LTO linking')
    test_deps += py.extension_module(
      'test_abi_multi_lto_ext',
      abi_multi_srcs,
      cpp_args: ext_compile_args + frontend_defs
        + ['-DNB_ABI_MULTI_NAME=test_abi_multi_lto_ext', '-DNB_ABI_MULTI_NS=abi_multi_lto'],
      override_options: ['b_lto=true'],
      kwargs: ext_kwargs,
    )
  endif

  # Points at a backend module that does not exist; its import must fail with
  # an instructive error message.
  test_deps += py.extension_module(
    'test_abi_missing_ext',
    'test_abi_missing.cpp',
    cpp_args: ext_compile_args + ['-DNB_BACKEND_MODULE=nanobind_backend_absent'],
    kwargs: ext_kwargs,
  )
endif

# ---------------------------------------------------------------------------
# Copy the Python test inputs next to the built extensions
# ---------------------------------------------------------------------------

test_support_files = [
  'common.py', 'conftest.py', 'py_stub_test.py',
  'pattern_file.nb', 'pattern_file_recursive.nb',
  'test_abi.py', 'test_accessor.py', 'test_callbacks.py', 'test_chrono.py',
  'test_classes.py', 'test_eigen.py', 'test_eigen_tensor.py', 'test_enum.py',
  'test_eval.py', 'test_exception.py', 'test_foreign.py', 'test_functions.py',
  'test_holders.py', 'test_inter_module.py', 'test_intrusive.py',
  'test_issue.py', 'test_jax.py', 'test_make_iterator.py', 'test_ndarray.py',
  'test_specialization.py', 'test_stl.py', 'test_stl_bind_map.py',
  'test_stl_bind_vector.py', 'test_stubgen.py', 'test_stubs.py',
  'test_tensorflow.py', 'test_thread.py', 'test_typing.py',
  # Stub reference files
  'test_classes_ext.pyi.ref', 'test_functions_ext.pyi.ref',
  'test_make_iterator_ext.pyi.ref', 'test_ndarray_ext.pyi.ref',
  'test_jax_ext.pyi.ref', 'test_tensorflow_ext.pyi.ref',
  'test_stl_ext.pyi.ref', 'test_enum_ext.pyi.ref', 'test_typing_ext.pyi.ref',
  'py_stub_test.pyi.ref',
]

foreach f : test_support_files
  configure_file(input: f, output: f, copy: true)
endforeach

# test_stubgen.py imports stubgen.py from next to it
configure_file(input: stubgen_file, output: 'stubgen.py', copy: true)

# ---------------------------------------------------------------------------
# Stub generation
# ---------------------------------------------------------------------------

build_tests_dir = meson.current_build_dir()

stub_names = [
  'functions', 'classes', 'ndarray', 'jax', 'tensorflow', 'stl', 'enum',
  'make_iterator',
]

foreach name : stub_names
  test_deps += custom_target(
    name + '_ext_stub',
    output: 'test_' + name + '_ext.pyi',
    command: [
      py, stubgen_file, '-q', '-i', build_tests_dir,
      '-m', 'test_' + name + '_ext', '-o', '@OUTPUT@',
    ],
    depends: [test_ext[name]] + stub_deps,
    build_by_default: true,
  )
endforeach

# The typing stub applies a pattern file, emits a py.typed marker, and forces
# '...' for values.
test_deps += custom_target(
  'typing_ext_stub',
  output: ['test_typing_ext.pyi', 'py.typed'],
  command: [
    py, stubgen_file, '-q', '--exclude-values',
    '-p', files('pattern_file.nb'), '-M', '@OUTPUT1@',
    '-i', build_tests_dir, '-m', 'test_typing_ext', '-o', '@OUTPUT0@',
  ],
  depends: [test_ext['typing']] + stub_deps,
  build_by_default: true,
)

# A stub for a pure-Python module
test_deps += custom_target(
  'py_stub',
  output: 'py_stub_test.pyi',
  command: [
    py, stubgen_file, '-q', '-i', build_tests_dir,
    '-m', 'py_stub_test', '-o', '@OUTPUT@',
  ],
  depends: [test_ext['stl']],
  build_by_default: true,
)

subdir('py_recursive_stub_test')
subdir('test_prefix_module')

# ---------------------------------------------------------------------------
# Run the test suite
# ---------------------------------------------------------------------------

test(
  'pytest',
  py,
  args: ['-m', 'pytest', build_tests_dir],
  workdir: build_tests_dir,
  depends: test_deps,
  timeout: 0,
)
