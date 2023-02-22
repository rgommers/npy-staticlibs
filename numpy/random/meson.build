# Build npyrandom library
# -----------------------
npyrandom_sources = [
  'src/distributions/logfactorial.c',
  'src/distributions/distributions.c',
  'src/distributions/random_mvhg_count.c',
  'src/distributions/random_mvhg_marginals.c',
  'src/distributions/random_hypergeometric.c',
]

npyrandom_lib = static_library('npyrandom',
  npyrandom_sources,
  c_args: staticlib_cflags,
  dependencies: [py_dep, np_core_dep],
)

libnpyrandom_dep = declare_dependency(
  link_with: [npyrandom_lib, npymath_lib],
  include_directories: '../core/include/numpy/random',
)
