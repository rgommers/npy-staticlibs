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
  # include_directories: '../core/include',
  dependencies: [py_dep, np_core_dep],
  install: true,
  install_dir: np_dir / 'random/lib',
)
