if get_option('kernel') == 'linux'
    kconfig = []

    if get_option('engine') in ['asbestos', 'unicorn']
        kconfig += 'CONFIG_ISH_EMULATOR_NONE=y'
    else
        error('Engine "' + get_option('engine') + '" not supported for linux kernel')
    endif

    kconfig += get_option('kconfig')
    meson_fragment = configure_file(
        command: [
            find_program('kconfig-fragment.sh'),
            '@OUTPUT@',
        ] + kconfig,
        output: 'linux-meson.config',
    )

    build_linux = custom_target('linux',
        command: [
            'env',
            'HOSTCC='+ ' '.join(meson.get_compiler('c', native: true).cmd_array() + get_option('c_args')),
            'CC='+ ' '.join(meson.get_compiler('c', native: false).cmd_array() + get_option('c_args')),
            # hack to find the right directories because meson is bad
            'ISH_SRC='+ meson.project_source_root(),
            'ISH_BUILD='+ meson.project_build_root(),
            'KCONFIG_FRAGMENTS='+ ' '.join([
                meson.project_source_root() / 'deps/linux.config',
                meson.project_build_root() / 'deps/linux-meson.config',
            ]),
            find_program('linux-build.sh'),
            '@OUTPUT0@',
            '@SOURCE_ROOT@/deps/linux',
            '@OUTPUT1@',
            '@DEPFILE@',
        ],
        depend_files: [
            'makefilter.py',
            'linux.config',
            meson_fragment,
        ],
        console: true,
        depfile: 'linux.d',
        output: ['liblinux.a', 'linux'])

    # Create empty include directories so the liblinux declaration doesn't throw File Not Found
    run_command('mkdir', [
        '-p',
        meson.current_build_dir()+'/linux/include/generated/uapi',
        meson.current_build_dir()+'/linux/arch/ish/include/generated/uapi'],
        check: false)

    linux_headers = declare_dependency(
        include_directories: [include_directories(
            'linux/arch/ish/include',
            'linux/arch/ish/include/generated',
            'linux/include',
            'linux/arch/ish/include/uapi',
            'linux/arch/ish/include/generated/uapi',
            'linux/include/uapi',
            'linux/include/generated/uapi',
        )] + includes,
        compile_args: [
            # '-nostdlibinc',
            '-include', 'linux/compiler_attributes.h',
            '-include', 'linux/kconfig.h',
            '-D__KERNEL__', '-U__weak',
            # Warnings triggered by kernel headers. Some of these
            # warn of UB, but I feel all right with silencing them
            # because if clang does something to break Linux
            # there'll be hell to pay.
            '-Wno-unused-parameter',
            '-Wno-sign-compare',
            '-Wno-implicit-fallthrough',
            '-Wno-pointer-sign',
            '-Wno-null-pointer-arithmetic',
        ],
        link_with: [build_linux[0]],
        sources: [build_linux[1]],
    )

    user_linux_headers = declare_dependency(
        include_directories: include_directories(
            'linux/arch/ish/include',
            'linux/include',
        ),
        compile_args: [
            '-include', 'user.h',
            '-include', 'linux/kconfig.h',
        ],
        sources: [build_linux[1]],
    )

    if host_machine.system() == 'darwin'
        liblinux_link_args = [
            '-Wl,-ld_classic',
            '-sectalign', '__DATA', '__percpu_first', '1000',
            '-sectalign', '__DATA', '__tracepoints', '20',
        ]
    elif host_machine.system() == 'linux'
        liblinux_link_args = [
            '-Wl,-T', meson.current_build_dir()+'/linux/arch/ish/kernel/vmlinux.lds'
        ]
    endif
    liblinux = declare_dependency(
        link_whole: [build_linux[0]],
        sources: [build_linux[1]],
        link_args: liblinux_link_args,
    )
endif
