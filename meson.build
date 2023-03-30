project('bluealsa-simple-handsfree', version: '0.0.1', license: 'MIT')

prefix = get_option('prefix')
assert(prefix.startswith('/'), 'Prefix is not absolute: "@0@"'.format(prefix))

bindir = join_paths(prefix, get_option('bindir'))
datadir = join_paths(prefix, get_option('datadir'))
unitdir = get_option('unitdir')
if unitdir == ''
	unitdir = join_paths(prefix, 'lib', 'systemd', 'system')
endif

conf_data = configuration_data()
conf_data.set('version', meson.project_version())
conf_data.set('prefix', prefix)
conf_data.set('bindir', bindir)
conf_data.set('datadir', datadir)

scripts = [
	'bluealsa-handsfree-audio.bash',
	'bluealsa-handsfree-control.bash',
	'bluealsa-simple-agent.bash',
	'bluealsa-handsfree',
]

units = [
	'bluealsa-handsfree-audio.service',
	'bluealsa-handsfree-control.service',
	'bluealsa-handsfree-control.socket',
	'bluealsa-simple-agent.service',
	'bluealsa-simple-pairable.service',
]

foreach script : scripts
	configure_file(
		input : script + '.in',
		output : script,
		configuration : conf_data,
		install: true,
		install_dir: bindir,
		install_mode: ['rwxr-xr-x', 'root', 'root'],
	)
endforeach

foreach unit : units
	configure_file(
		input : unit + '.in',
		output : unit,
		configuration : conf_data,
		install: true,
		install_dir: unitdir,
		install_mode: ['rw-r--r--', 'root', 'root'],
	)
endforeach

install_data('ring.wav')
