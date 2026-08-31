# Copied into the build tree so stubgen can import the package and 'pytest' can
# compare the generated stubs against the reference files next to them.
foreach f : [
  '__init__.py', 'prefix.py', 'prefixabc.py',
  '__init__.pyi.ref', 'prefix.pyi.ref', 'prefixabc.pyi.ref',
]
  configure_file(input: f, output: f, copy: true)
endforeach

test_deps += custom_target(
  'test_prefix_module_stub',
  output: '__init__.pyi',
  command: [
    py, stubgen_file, '-q', '-r',
    '-i', build_tests_dir, '-m', 'test_prefix_module',
    '-O', meson.current_build_dir(),
  ],
  depends: stub_deps,
  build_by_default: true,
)
