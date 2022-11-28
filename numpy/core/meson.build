# This file should contain what setup.py + setup_common.py do (WIP)
#
# Potential issues to address or keep track of:
#   - sincos detection incorrect on NetBSD: https://github.com/mesonbuild/meson/issues/10641

# Versioning support
#-------------------
#
# How to change C_API_VERSION ?
#   - increase C_API_VERSION value
#   - record the hash for the new C API with the cversions.py script
#   and add the hash to cversions.txt
# The hash values are used to remind developers when the C API number was not
# updated - generates a MismatchCAPIWarning warning which is turned into an
# exception for released version.

# Binary compatibility version number. This number is increased whenever the
# C-API is changed such that binary compatibility is broken, i.e. whenever a
# recompile of extension modules is needed.
C_ABI_VERSION = '0x01000009'

# Minor API version.  This number is increased whenever a change is made to the
# C-API -- whether it breaks binary compatibility or not.  Some changes, such
# as adding a function pointer to the end of the function table, can be made
# without breaking binary compatibility.  In this case, only the C_API_VERSION
# (*not* C_ABI_VERSION) would be increased.  Whenever binary compatibility is
# broken, both C_API_VERSION and C_ABI_VERSION should be increased.
#
# The version needs to be kept in sync with that in cversions.txt.
#
# 0x00000008 - 1.7.x
# 0x00000009 - 1.8.x
# 0x00000009 - 1.9.x
# 0x0000000a - 1.10.x
# 0x0000000a - 1.11.x
# 0x0000000a - 1.12.x
# 0x0000000b - 1.13.x
# 0x0000000c - 1.14.x
# 0x0000000c - 1.15.x
# 0x0000000d - 1.16.x
# 0x0000000d - 1.19.x
# 0x0000000e - 1.20.x
# 0x0000000e - 1.21.x
# 0x0000000f - 1.22.x
# 0x00000010 - 1.23.x
# 0x00000010 - 1.24.x
C_API_VERSION = '0x00000010'

# Check whether we have a mismatch between the set C API VERSION and the
# actual C API VERSION. Will raise a MismatchCAPIError if so.
run_command('code_generators/verify_c_api_version.py', '--api-version', C_API_VERSION, check: true)


# Generate config.h and _numpyconfig.h
# ------------------------------------
#
# There are two generated headers:
#   - config.h: a private header, which is not installed and used only at
#               build time, mostly to determine whether or not optional
#               headers, compiler attributes, etc. are available
#   - _numpyconfig.h: a header with public `NPY_` symbols which get made
#                     available via numpyconfig.h
#
# Note that `HAVE_*` symbols indicate the presence or absence of a checked
# property of the build environment. When available, these symbols get defined
# to `1`; when not available they must not be defined at all. This is
# important, because code in NumPy typically does not check the value but only
# whether the symbol is defined. So `#define HAVE_SOMETHING 0` is wrong.

cdata = configuration_data()

cdata.set('NPY_ABI_VERSION', C_ABI_VERSION)
cdata.set('NPY_API_VERSION', C_API_VERSION)

use_svml = (
  host_machine.system() == 'linux' and
  host_machine.cpu_family() == 'x86_64' and
  not get_option('disable-svml')
)
if use_svml
  cdata.set10('NPY_CAN_LINK_SVML', true)
  if not fs.exists('src/umath/svml')
    error('Missing the `SVML` git submodule! Run `git submodule update --init` to fix this.')
  endif
endif

# Check sizes of types. Note, some of these landed in config.h before, but were
# unused. So clean that up and only define the NPY_SIZEOF flavors rather than
# the SIZEOF ones
types_to_check = [
  ['NPY_SIZEOF_SHORT', 'short'],
  ['NPY_SIZEOF_INT', 'int'],
  ['NPY_SIZEOF_LONG', 'long'],
  ['NPY_SIZEOF_LONGLONG', 'long long'],
  ['NPY_SIZEOF_FLOAT', 'float'],
  ['NPY_SIZEOF_DOUBLE', 'double'],
  ['NPY_SIZEOF_LONGDOUBLE', 'long double'],
]
foreach symbol_type: types_to_check
  cdata.set(symbol_type[0], cc.sizeof(symbol_type[1]))
endforeach
cdata.set('NPY_SIZEOF_OFF_T', cc.sizeof('off_t', prefix: '#include <sys/types.h>'))
cdata.set('NPY_SIZEOF_PY_INTPTR_T',
  cc.sizeof('Py_intptr_t', dependencies: py_dep, prefix: '#include <Python.h>'))
cdata.set('NPY_SIZEOF_PY_LONG_LONG',
  cc.sizeof('PY_LONG_LONG', dependencies: py_dep, prefix: '#include <Python.h>'))

# Check for complex support
if cc.has_header('complex.h')
  cdata.set10('HAVE_COMPLEX_H', true)
  cdata.set10('NPY_USE_C99_COMPLEX', true)
  complex_types_to_check = [
    ['NPY_HAVE_COMPLEX_FLOAT', 'NPY_SIZEOF_COMPLEX_FLOAT', 'complex float', 'float'],
    ['NPY_HAVE_COMPLEX_DOUBLE', 'NPY_SIZEOF_COMPLEX_DOUBLE', 'complex double', 'double'],
    ['NPY_HAVE_COMPLEX_LONG_DOUBLE', 'NPY_SIZEOF_COMPLEX_LONGDOUBLE', 'complex long double', 'long double'],
  ]
  foreach symbol_type: complex_types_to_check
    if cc.has_type(symbol_type[2], prefix: '#include <complex.h>')
      cdata.set10(symbol_type[0], true)
      # Determine size of NumPy's own, struct-based, complex types. Binary
      # compatibility with C99 complex types is checked at build time in `npy_common.h`.
      ftype = symbol_type[3]
      cdata.set(symbol_type[1], cc.sizeof(f'struct {@ftype@ __x; @ftype@ __y;}'))
    endif
  endforeach
endif

# Mandatory functions: if not found, fail the build
# Some of these can still be blocklisted if the C99 implementation
# is buggy, see numpy/core/src/common/npy_config.h
mandatory_funcs = [
  'sin', 'cos', 'tan', 'sinh', 'cosh', 'tanh', 'fabs',
  'floor', 'ceil', 'sqrt', 'log10', 'log', 'exp', 'asin',
  'acos', 'atan', 'fmod', 'modf', 'frexp', 'ldexp',
  'expm1', 'log1p', 'acosh', 'asinh', 'atanh',
  'rint', 'trunc', 'exp2', 
  'copysign', 'nextafter', 'strtoll', 'strtoull', 'cbrt',
  'log2', 'pow', 'hypot', 'atan2',
  'csin', 'csinh', 'ccos', 'ccosh', 'ctan', 'ctanh',
  'creal', 'cimag', 'conj'
]
foreach func: mandatory_funcs
  if not cc.has_function(func)
    error('Function `{func}` not found')
  endif
endforeach

c99_complex_funcs = [
  'cabs', 'cacos', 'cacosh', 'carg', 'casin', 'casinh', 'catan',
  'catanh', 'cexp', 'clog', 'cpow', 'csqrt'
]
foreach func: c99_complex_funcs
  func_single = func + 'f'
  func_longdouble = func + 'l'
  if cc.has_function(func)
    cdata.set10('HAVE_' + func.to_upper(), true)
  endif
  if cc.has_function(func_single)
    cdata.set10('HAVE_' + func_single.to_upper(), true)
  endif
  if cc.has_function(func_longdouble)
    cdata.set10('HAVE_' + func_longdouble.to_upper(), true)
  endif
endforeach

# We require C99 so these should always be found at build time. But for
# libnpymath as a C99 compat layer, these may still be relevant.
c99_macros = ['isfinite', 'isinf', 'isnan', 'signbit']
foreach macro: c99_macros
  if cc.has_function(macro)
    cdata.set10('NPY_HAVE_DECL_' + macro.to_upper(), true)
    if not cc.has_header_symbol('Python.h', macro, dependencies: py_dep)
      # Add in config.h as well, except if it will clash with Python's config.h content
      cdata.set10('HAVE_DECL_' + macro.to_upper(), true)
    endif
  endif
endforeach

# variable attributes tested via "int %s a" % attribute
optional_variable_attributes = [
  ['__thread', 'HAVE__THREAD'],
  ['__declspec(thread)', 'HAVE___DECLSPEC_THREAD_']
]
foreach optional_attr: optional_variable_attributes
  attr = optional_attr[0]
  code = f'''
    #pragma GCC diagnostic error "-Wattributes"
    #pragma clang diagnostic error "-Wattributes"

    int @attr@ foo;
  '''
  code += '''
    int
    main()
    {
      return 0;
    }
  '''
  if cc.compiles(code)
    cdata.set10(optional_attr[1], true)
  endif
endforeach

inc_curdir = include_directories('.')
optional_file_funcs = ['fallocate', 'ftello', 'fseeko']
foreach filefunc_maybe: optional_file_funcs
  config_value = 'HAVE_' + filefunc_maybe.to_upper()
  # Some functions may already have HAVE_* defined by `Python.h`. Python puts
  # its config.h in the public Python.h namespace, so we have a possible clash
  # for the common functions we test. Hence we skip those checks.
  if (filefunc_maybe == 'fallocate' or
      not cc.has_header_symbol('Python.h', config_value, dependencies: py_dep)
    )
    if cc.has_function(filefunc_maybe,
                       include_directories: inc_curdir,
                       prefix: '#include "feature_detection_stdio.h"'
      )
      cdata.set10(config_value, true)
    endif
  endif
endforeach

# Optional locale function
have_strtold_l = cc.has_function('strtold_l', include_directories: inc_curdir,
  prefix:'''
  #include <stdlib.h>
  #include <xlocale.h>
  #include "feature_detection_locale.h"
''')
if not have_strtold_l
  # Retry with locale.h, seems to vary across Linux distros
  have_strtold_l = cc.has_function('strtold_l', include_directories: inc_curdir,
    prefix:'''
    #include <stdlib.h>
    #include <locale.h>
    #include "feature_detection_locale.h"
  ''')
endif
if have_strtold_l
  cdata.set10('HAVE_STRTOLD_L', true)
else
  # FIXME: this is wrong! the HAVE_ define should not exist, or it'll be
  # interpreted as the function being available (true/false does nothing, see
  # note on HAVE_ defines higher up). This is necessary though in order to make
  # the Linux CI job pass. So either the check is wrong somehow, or this
  # function is not available in CI. For the latter there is a fallback path,
  # but that is broken because we don't have the exact long double
  # representation checks.
  cdata.set10('HAVE_STRTOLD_L', false)
endif

# Other optional functions
optional_misc_funcs = [
  'backtrace',
  'madvise',
]
foreach func: optional_misc_funcs
  if cc.has_function(func,
      include_directories: inc_curdir,
      prefix: '#include "feature_detection_misc.h"'
    )
    cdata.set10('HAVE_' + func.to_upper(), true)
  endif
endforeach

# SSE headers only enabled automatically on amd64/x32 builds
optional_headers = [
  'xmmintrin.h',  # SSE
  'emmintrin.h',  # SSE2
  'immintrin.h',  # AVX
  'features.h',   # for glibc version linux
  'xlocale.h',    # see GH#8367
  'dlfcn.h',      # dladdr
  'execinfo.h',   # backtrace
  'libunwind.h',  # backtrace for LLVM/Clang using libunwind
  'sys/mman.h',   # madvise
]
foreach header: optional_headers
  if cc.has_header(header)
    cdata.set10('HAVE_' + header.to_upper().replace('.', '_').replace('/', '_'), true)
  endif
endforeach

# Optional compiler attributes
# TODO: this doesn't work with cc.has_function_attribute, see
#       https://github.com/mesonbuild/meson/issues/10732
optional_function_attributes = [
  ['optimize("unroll-loops")', 'OPTIMIZE_UNROLL_LOOPS'],
  ['optimize("O3")', 'OPTIMIZE_OPT_3'],
  ['optimize("O2")', 'OPTIMIZE_OPT_2'],
  ['optimize("nonnull (1)")', 'NONNULL'],
  ]
if host_machine.cpu_family() in ['x86', 'x86_64']
  optional_function_attributes += [
    ['target("avx")', 'TARGET_AVX'],
    ['target("avx2")', 'TARGET_AVX2'],
    ['target("avx512f")', 'TARGET_AVX512F'],
    ['target("avx512f,avx512dq,avx512bw,avx512vl,avx512cd")', 'TARGET_AVX512_SKX'],
  ]
  # TODO: add the _WITH_INTRINSICS_AVX list
endif
#foreach attr: optional_function_attributes
#  if cc.has_function_attribute(attr[0])
#    cdata.set10('HAVE_ATTRIBUTE_' + attr[1], true)
#  endif
#endforeach

# Optional GCC compiler builtins and their call arguments.
# If given, a required header and definition name (HAVE_ prepended)
# Call arguments are required as the compiler will do strict signature checking
optional_intrinsics = [
  ['__builtin_isnan', '5.', [], []],
  ['__builtin_isinf', '5.', [], []],
  ['__builtin_isfinite', '5.', [], []],
  ['__builtin_bswap32', '5u', [], []],
  ['__builtin_bswap64', '5u', [], []],
  ['__builtin_expect', '5, 0', [], []],
  ['__builtin_mul_overflow', '5, 5, (int*)5', [], []],
]
if host_machine.cpu_family() in ['x86', 'x86_64']
 optional_intrinsics += [
    # MMX only needed for icc, but some clang's don't have it
    ['_m_from_int64', '0', ['emmintrin.h'], []],
    ['_mm_load_ps', '(float*)0', ['xmmintrin.h'], []],  # SSE
    ['_mm_prefetch', '(float*)0, _MM_HINT_NTA', ['xmmintrin.h'], []],  # SSE
    ['_mm_load_pd', '(double*)0', ['emmintrin.h'], []],  # SSE2
    ['__builtin_prefetch', '(float*)0, 0, 3', [], []],
    # Check that the linker can handle AVX
    ['__asm__ volatile', '"vpand %xmm1, %xmm2, %xmm3"', ['stdio.h'], ['HAVE_LINK_AVX']],
    ['__asm__ volatile', '"vpand %ymm1, %ymm2, %ymm3"', ['stdio.h'], ['HAVE_LINK_AVX2']],
    ['__asm__ volatile', '"vpaddd %zmm1, %zmm2, %zmm3"', ['stdio.h'], ['HAVE_LINK_AVX512F']],
    ['__asm__ volatile',
     '"vfpclasspd $0x40, %zmm15, %k6\\n vmovdqu8 %xmm0, %xmm1\\n vpbroadcastmb2q %k0, %xmm0"',
     ['stdio.h'], ['HAVE_LINK_AVX512_SKX']
    ],
    ['__asm__ volatile', '"xgetbv"', ['stdio.h'], ['HAVE_XGETBV']],
  ]
endif
foreach intrin: optional_intrinsics
  func = intrin[0]
  func_args = intrin[1]
  header = intrin[2]
  define = intrin[3]
  code = ''
  if header.length() == 1
    header_name = header[0]
    code += f'#include <@header_name@>'
  endif
  code += f'''
    #ifdef _MSC_VER
    #pragma function(@func@)
    #endif
    int main(void) {
      @func@(@func_args@);
      return 0;
    };
    '''
  if define.length() == 1
    define_name = define[0]
  else
    define_name = 'HAVE_' + func.to_upper()
  endif
  if cc.links(code)
    cdata.set10(define_name, true)
  endif
endforeach

# long double representation detection (see setup_common.py)
# TODO: this is still incomplete, and different from how it's done in the
# numpy.distutils based build, see https://github.com/mesonbuild/meson/issues/11068
longdouble_size = cc.sizeof('long double')
if longdouble_size == 8
  if host_machine.endian() == 'little'
    longdouble_format = 'IEEE_DOUBLE_LE'
  else
    longdouble_format = 'IEEE_DOUBLE_BE'
  endif
elif longdouble_size == 12
  error('This should not be possible, 12 bits of "content" should still result in sizeof() being 16. Please report this error!'
  )
elif longdouble_size == 16
  if host_machine.endian() == 'little'
    # FIXME: this varies, there's multiple formats here! Not yet implemented.
    #        TBD how we deal with the mess of old long double formats.
    longdouble_format = 'INTEL_EXTENDED_16_BYTES_LE'
  else
    error('No idea what this is ....')
  endif
else
  error('Unknown long double size: ' + londouble_size)
endif
cdata.set10('HAVE_LDOUBLE_' + longdouble_format, true)

if cc.has_header('endian.h')
  cdata.set10('NPY_HAVE_ENDIAN_H', true)
endif
if cc.has_header('sys/endian.h')
  cdata.set10('NPY_HAVE_SYS_ENDIAN_H', true)
endif
if is_windows
  cdata.set10('NPY_NO_SIGNAL', true)
endif
# Command-line switch; distutils build checked for `NPY_NOSMP` env var instead
# TODO: document this (search for NPY_NOSMP in C API docs)
cdata.set10('NPY_NO_SMP', get_option('disable-threading'))

# Use bogus stride debug aid to flush out bugs where users use strides of
# dimensions with length 1 to index a full contiguous array.
cdata.set10('NPY_RELAXED_STRIDES_DEBUG', get_option('relaxed-strides-debug'))

# Check whether we can use inttypes (C99) formats
if cc.has_header_symbol('inttypes.h', 'PRIdPTR')
  cdata.set10('NPY_USE_C99_FORMATS', true)
endif

visibility_hidden = ''
if cc.has_function_attribute('visibility:hidden')
  visibility_hidden = '__attribute__((visibility("hidden")))'
endif
cdata.set('NPY_VISIBILITY_HIDDEN', visibility_hidden)


config_h = configure_file(
  input: 'config.h.in',
  output: 'config.h',
  configuration: cdata,
  install: false
)

_numpyconfig_h = configure_file(
  input: 'include/numpy/_numpyconfig.h.in',
  output: '_numpyconfig.h',
  configuration: cdata,
  install: true,
  install_dir: np_dir / 'core/include/numpy'
)

# Build npymath static library
# ----------------------------

staticlib_cflags = []
if cc.get_id() == 'msvc'
  # Disable voltbl section for vc142 to allow link using mingw-w64; see:
  # https://github.com/matthew-brett/dll_investigation/issues/1#issuecomment-1100468171
  # Needs to be added to static libraries that are shipped for reuse (i.e.,
  # libnpymath and libnpyrandom)
  if cc.has_argument('-d2VolatileMetadata-')
     staticlib_cflags +=  '-d2VolatileMetadata-'
   endif
endif

npy_math_internal_h = custom_target(
  output: 'npy_math_internal.h',
  input: 'src/npymath/npy_math_internal.h.src',
  command: [src_file_cli, '@INPUT@', '-o', '@OUTPUT@'],
)

npymath_sources = [
  src_file.process('src/npymath/ieee754.c.src'),
  src_file.process('src/npymath/npy_math_complex.c.src'),
  npy_math_internal_h,
  'src/npymath/_signbit.c',
  'src/npymath/halffloat.c',
  'src/npymath/npy_math.c',
]
npymath_lib = static_library('npymath',
  npymath_sources,
  c_args: staticlib_cflags,
  include_directories: ['include', 'src/npymath', 'src/common'],
  dependencies: py_dep,
  install: true,
  install_dir: np_dir / 'core/lib',
)

dir_separator = '/'
if build_machine.system() == 'windows'
  dir_separator = '\\'
endif
configure_file(
  input: 'npymath.ini.in',
  output: 'npymath.ini',
  configuration: configuration_data({
    'pkgname' : 'numpy.core',
    'sep' : dir_separator,
  }),
  install: true,
  install_dir: np_dir / 'core/lib/npy-pkg-config'
)
configure_file(
  input: 'mlib.ini.in',
  output: 'mlib.ini',
  configuration: configuration_data({
    'posix_mathlib' : mlib_linkflag,
    'msvc_mathlib' : 'm.lib',
  }),
  install: true,
  install_dir: np_dir / 'core/lib/npy-pkg-config'
)

if false
  # This doesn't quite work (yet), it assumes we'll install headers under
  # include/, and trying to add the correct path with `extra_cflags` runs into
  # not being able to get a path relative to the prefix.
  # Note that installing numpy headers under include/ would be a good idea, but
  # that needs work (and may run into trouble with wheels perhaps, not quite
  # clear if they allow this).
  pkg = import('pkgconfig')
  pkg.generate(npymath_lib,
    name: 'npymath',
    description: 'Portable, core math library implementing C99 standard',
    url: 'https://github.com/numpy/numpy',
    requires: 'numpy',
    install_dir: np_dir / 'core/lib/npy-pkg-config',
    extra_cflags: '-I${includedir}' + np_dir / 'core' / 'include',
  )
endif

# Generate NumPy C API sources
# ----------------------------

# This is a single C file. It needs to be built before _multiarray_umath starts
# building, but we can't add it to the sources of that extension (this C file
# doesn't compile, it's only included in another r file. Hence use the slightly
# hacky --ignore argument to the next custom_target().
src_umath_api_c = custom_target('__umath_generated',
  output : '__umath_generated.c',
  input : 'code_generators/generate_umath.py',
  command: [py, '@INPUT@', '-o', '@OUTPUT@'],
)

src_umath_doc_h = custom_target('_umath_doc_generated',
  output : '_umath_doc_generated.h',
  input : 'code_generators/generate_umath_doc.py',
  command: [py, '@INPUT@', '-o', '@OUTPUT@'],
)

src_numpy_api = custom_target('__multiarray_api',
  output : ['__multiarray_api.c', '__multiarray_api.h', 'multiarray_api.txt'],
  input : 'code_generators/generate_numpy_api.py',
  command: [py, '@INPUT@', '-o', '@OUTDIR@', '--ignore', src_umath_api_c],
  install: true,  # NOTE: setup.py build installs all, but just need .h?
  install_dir: np_dir / 'core/include/numpy'
)

src_ufunc_api = custom_target('__ufunc_api',
  output : ['__ufunc_api.c', '__ufunc_api.h', 'ufunc_api.txt'],
  input : 'code_generators/generate_ufunc_api.py',
  command: [py, '@INPUT@', '-o', '@OUTDIR@'],
  install: true,  # NOTE: setup.py build installs all, but just need .h?
  install_dir: np_dir / 'core/include/numpy'
)


# Set common build flags for C and C++ code
# -----------------------------------------

# TODO: change to "feature" option in meson_options.txt? See
# https://mesonbuild.com/Build-options.html#build-options
disable_simd_optimizations = []
if get_option('disable-simd-optimizations')
  disable_simd_optimizations = '-DNPY_DISABLE_OPTIMIZATION'
endif

# Common build flags
c_args_common = [
  '-DNPY_INTERNAL_BUILD',
  '-DHAVE_NPY_CONFIG_H',
  disable_simd_optimizations,
  cflags_large_file_support,
]

# Same as NPY_CXX_FLAGS (TODO: extend for what ccompiler_opt adds)
cpp_args_common = c_args_common + [
  '-D__STDC_VERSION__=0',  # for compatibility with C headers
  '-fno-exceptions',  # no exception support
  '-fno-rtti',  # no runtime type information
]

# Other submodules depend on generated headers and include directories from
# core, wrap those up into a reusable dependency. Also useful for some test
# modules in this build file.
np_core_dep = declare_dependency(
  sources: [
    _numpyconfig_h,
    npy_math_internal_h,
    src_numpy_api[1],  # __multiarray_api.h
    src_ufunc_api[1],  # __ufunc_api.h
  ],
  include_directories: [
    '.',
    'include',
    'src/common',
  ]
)


# Build multiarray_tests module
# -----------------------------
py.extension_module('_multiarray_tests',
  [
    src_file.process('src/multiarray/_multiarray_tests.c.src'),
    'src/common/mem_overlap.c',
    'src/common/npy_argparse.c',
    'src/common/npy_hashtable.c',
    src_file.process('src/common/templ_common.h.src')
  ],
  c_args: c_args_common,
  include_directories: ['src/multiarray', 'src/npymath'],
  dependencies: np_core_dep,
  link_with: npymath_lib,
  gnu_symbol_visibility: 'default',
  install: true,
  subdir: 'numpy/core',
)

test_modules_src = [
  ['_umath_tests', [
      src_file.process('src/umath/_umath_tests.c.src'),
      'src/umath/_umath_tests.dispatch.c',
      'src/common/npy_cpu_features.c',
    ]],
  ['_rational_tests', 'src/umath/_rational_tests.c'],
  ['_struct_ufunc_tests', 'src/umath/_struct_ufunc_tests.c'],
  ['_operand_flag_tests', 'src/umath/_operand_flag_tests.c'],
]
foreach gen: test_modules_src
  py.extension_module(gen[0],
    gen[1],
    c_args: c_args_common,
    include_directories: ['src/multiarray', 'src/npymath'],
    dependencies: np_core_dep,
    install: true,
    subdir: 'numpy/core',
  )
endforeach

# Build _multiarray_umath module
# ------------------------------
src_multiarray_umath_common = [
  'src/common/array_assign.c',
  'src/common/mem_overlap.c',
  'src/common/npy_argparse.c',
  'src/common/npy_hashtable.c',
  'src/common/npy_longdouble.c',
  'src/common/ucsnarrow.c',
  'src/common/ufunc_override.c',
  'src/common/numpyos.c',
  'src/common/npy_cpu_features.c',
  src_file.process('src/common/templ_common.h.src')
]
if have_blas
  src_multiarray_umath_common += [
    'src/common/cblasfuncs.c',
    'src/common/python_xerbla.c',
  ]
endif

src_multiarray = [
  'src/multiarray/abstractdtypes.c',
  'src/multiarray/alloc.c',
  src_file.process('src/multiarray/argfunc.dispatch.c.src'),
  'src/multiarray/arrayobject.c',
  src_file.process('src/multiarray/arraytypes.h.src'),
  'src/multiarray/array_coercion.c',
  'src/multiarray/array_method.c',
  'src/multiarray/array_assign_scalar.c',
  'src/multiarray/array_assign_array.c',
  'src/multiarray/arrayfunction_override.c',
  src_file.process('src/multiarray/arraytypes.c.src'),
  'src/multiarray/buffer.c',
  'src/multiarray/calculation.c',
  'src/multiarray/compiled_base.c',
  'src/multiarray/common.c',
  'src/multiarray/common_dtype.c',
  'src/multiarray/convert.c',
  'src/multiarray/convert_datatype.c',
  'src/multiarray/conversion_utils.c',
  'src/multiarray/ctors.c',
  'src/multiarray/datetime.c',
  'src/multiarray/datetime_strings.c',
  'src/multiarray/datetime_busday.c',
  'src/multiarray/datetime_busdaycal.c',
  'src/multiarray/descriptor.c',
  'src/multiarray/dlpack.c',
  'src/multiarray/dtypemeta.c',
  'src/multiarray/dragon4.c',
  'src/multiarray/dtype_transfer.c',
  src_file.process('src/multiarray/einsum.c.src'),
  src_file.process('src/multiarray/einsum_sumprod.c.src'),
  'src/multiarray/experimental_public_dtype_api.c',
  'src/multiarray/flagsobject.c',
  'src/multiarray/getset.c',
  'src/multiarray/hashdescr.c',
  'src/multiarray/item_selection.c',
  'src/multiarray/iterators.c',
  'src/multiarray/legacy_dtype_implementation.c',
  src_file.process('src/multiarray/lowlevel_strided_loops.c.src'),
  'src/multiarray/mapping.c',
  'src/multiarray/methods.c',
  'src/multiarray/multiarraymodule.c',
  'src/multiarray/nditer_api.c',
  'src/multiarray/nditer_constr.c',
  'src/multiarray/nditer_pywrap.c',
  src_file.process('src/multiarray/nditer_templ.c.src'),
  'src/multiarray/number.c',
  'src/multiarray/refcount.c',
  src_file.process('src/multiarray/scalartypes.c.src'),
  'src/multiarray/sequence.c',
  'src/multiarray/shape.c',
  'src/multiarray/scalarapi.c',
  'src/multiarray/strfuncs.c',
  'src/multiarray/temp_elide.c',
  'src/multiarray/typeinfo.c',
  'src/multiarray/usertypes.c',
  'src/multiarray/vdot.c',
  src_file.process('src/common/npy_sort.h.src'),
  'src/npysort/x86-qsort.dispatch.cpp',
  'src/npysort/quicksort.cpp',
  'src/npysort/mergesort.cpp',
  'src/npysort/timsort.cpp',
  'src/npysort/heapsort.cpp',
  'src/npysort/radixsort.cpp',
  'src/common/npy_partition.h',
  'src/npysort/selection.cpp',
  'src/common/npy_binsearch.h',
  'src/npysort/binsearch.cpp',
  'src/multiarray/textreading/conversions.c',
  'src/multiarray/textreading/field_types.c',
  'src/multiarray/textreading/growth.c',
  'src/multiarray/textreading/readtext.c',
  'src/multiarray/textreading/rows.c',
  'src/multiarray/textreading/stream_pyobject.c',
  'src/multiarray/textreading/str_to_int.c',
  'src/multiarray/textreading/tokenize.cpp',
]

src_umath = [
  src_file.process('src/umath/funcs.inc.src'),
  src_file.process('src/umath/loops.h.src'),
  src_file.process('src/umath/loops_utils.h.src'),
  src_file.process('src/umath/loops.c.src'),
  src_file.process('src/umath/loops_arithm_fp.dispatch.c.src'),
  src_file.process('src/umath/loops_arithmetic.dispatch.c.src'),
  src_file.process('src/umath/loops_comparison.dispatch.c.src'),
  src_file.process('src/umath/loops_exponent_log.dispatch.c.src'),
  src_file.process('src/umath/loops_hyperbolic.dispatch.c.src'),
  src_file.process('src/umath/loops_minmax.dispatch.c.src'),
  src_file.process('src/umath/loops_modulo.dispatch.c.src'),
  src_file.process('src/umath/loops_trigonometric.dispatch.c.src'),
  src_file.process('src/umath/loops_umath_fp.dispatch.c.src'),
  src_file.process('src/umath/loops_unary_fp.dispatch.c.src'),
  src_file.process('src/umath/matmul.c.src'),
  src_file.process('src/umath/matmul.h.src'),
  src_file.process('src/umath/simd.inc.src'),
  'src/umath/ufunc_type_resolution.c',
  'src/umath/clip.cpp',
  'src/umath/clip.h',
  'src/umath/dispatching.c',
  'src/umath/extobj.c',
  'src/umath/legacy_array_method.c',
  'src/umath/override.c',
  'src/umath/reduction.c',
  src_file.process('src/umath/scalarmath.c.src'),
  'src/umath/ufunc_object.c',
  'src/umath/umathmodule.c',
  'src/umath/string_ufuncs.cpp',
  'src/umath/wrapping_array_method.c',
  # For testing. Eventually, should use public API and be separate:
  'src/umath/_scaled_float_dtype.c',
]

# SVML object files. If functionality is migrated to universal intrinsics and
# the object files are no longer needed, comment out the relevant object files
# here. Note that this migration is desirable; we then get the performance
# benefits for all platforms rather than only for AVX512 on 64-bit Linux, and
# may be able to avoid the accuracy regressions in SVML.
svml_objects = []
if use_svml
  svml_objects += [
    'src/umath/svml/linux/avx512/svml_z0_acos_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_acos_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_acosh_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_acosh_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_asin_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_asin_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_asinh_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_asinh_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_atan2_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_atan2_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_atan_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_atan_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_atanh_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_atanh_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_cbrt_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_cbrt_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_cos_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_cos_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_cosh_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_cosh_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_exp2_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_exp2_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_exp_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_exp_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_expm1_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_expm1_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log10_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log10_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log1p_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log1p_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log2_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log2_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_log_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_pow_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_pow_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_sin_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_sin_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_sinh_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_sinh_s_la.s',
    'src/umath/svml/linux/avx512/svml_z0_tan_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_tan_s_la.s',
    # 'src/umath/svml/linux/avx512/svml_z0_tanh_d_la.s',
    'src/umath/svml/linux/avx512/svml_z0_tanh_s_la.s',
  ]
endif

py.extension_module('_multiarray_umath',
  [
    config_h,
    _numpyconfig_h,
    src_multiarray,
    src_multiarray_umath_common,
    src_umath,
    src_ufunc_api[1],  # __ufunc_api.h
    src_numpy_api[1],  # __multiarray_api.h
    src_umath_doc_h,
    npy_math_internal_h,
  ],
  objects: svml_objects,
  c_args: c_args_common,
  cpp_args: cpp_args_common,
  include_directories: [
    'include',
    'src/common',
    'src/multiarray',
    'src/npymath',
    'src/umath',
  ],
  dependencies: blas,
  link_with: npymath_lib,
  install: true,
  subdir: 'numpy/core',
)

# Build SIMD module
# -----------------

py.extension_module('_simd',
  [
    'src/common/npy_cpu_features.c',
    'src/_simd/_simd.c',
    src_file.process('src/_simd/_simd_inc.h.src'),
    src_file.process('src/_simd/_simd_data.inc.src'),
    src_file.process('src/_simd/_simd.dispatch.c.src'),
  ],
  c_args: c_args_common,
  #include_directories: ['src/multiarray', 'src/npymath'],
  include_directories: ['src/_simd'],
  dependencies: np_core_dep,
  install: true,
  subdir: 'numpy/core',
)

python_sources = [
  '__init__.py',
  '__init__.pyi',
  '_add_newdocs.py',
  '_add_newdocs_scalars.py',
  '_asarray.py',
  '_asarray.pyi',
  '_dtype.py',
  '_dtype_ctypes.py',
  '_exceptions.py',
  '_internal.py',
  '_internal.pyi',
  '_machar.py',
  '_methods.py',
  '_string_helpers.py',
  '_type_aliases.py',
  '_type_aliases.pyi',
  '_ufunc_config.py',
  '_ufunc_config.pyi',
  'arrayprint.py',
  'arrayprint.pyi',
  'cversions.py',
  'defchararray.py',
  'defchararray.pyi',
  'einsumfunc.py',
  'einsumfunc.pyi',
  'fromnumeric.py',
  'fromnumeric.pyi',
  'function_base.py',
  'function_base.pyi',
  'getlimits.py',
  'getlimits.pyi',
  'memmap.py',
  'memmap.pyi',
  'multiarray.py',
  'multiarray.pyi',
  'numeric.py',
  'numeric.pyi',
  'numerictypes.py',
  'numerictypes.pyi',
  'overrides.py',
  'records.py',
  'records.pyi',
  'shape_base.py',
  'shape_base.pyi',
  'umath.py',
  'umath_tests.py',
]

py.install_sources(
  python_sources,
  subdir: 'numpy/core'
)

subdir('include')
install_subdir('tests', install_dir: np_dir / 'core')