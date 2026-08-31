# Copied into the build tree so stubgen can import the package and 'pytest' can
# compare the generated stubs against the reference files next to them.
foreach f : [
  '__init__.py', 'alias.py', 'bar.py',
  '__init__.pyi.ref', 'alias.pyi.ref', 'bar.pyi.ref',
]
  configure_file(input: f, output: f, copy: true)
endforeach

subdir('sub')

test_deps += custom_target(
  'py_recursive_stub',
  output: '__init__.pyi',
  command: [
    py, stubgen_file, '-q', '-r',
    '-p', meson.project_source_root() / 'tests' / 'pattern_file_recursive.nb',
    '-i', build_tests_dir, '-m', 'py_recursive_stub_test',
    '-O', meson.current_build_dir(),
  ],
  depends: stub_deps,
  build_by_default: true,
)
