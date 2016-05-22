require 'active_support/core_ext/hash'
require 'active_support/core_ext/string'
require 'yaml'

build = '_build'

task default: :build
task build: [:reset_build_directory, :compile_overviews]
task travis: [:build, :zip]

task :reset_build_directory do
  FileUtils.remove_entry_secure build if Dir.exist? build
  FileUtils.mkdir_p build
end

task :compile_overviews do
  Dir['overviews/*.yml'].each do |path|
    overview = ActiveSupport::HashWithIndifferentAccess.new(YAML.load_file(path))
    name = File.basename(path, File.extname(path))
    overview[:tabs].each do |tab|
      overview[:tab] = tab
      compile_overview name, File.join(build, "#{name}-#{tab}.yaml"), overview
    end
  end
end

task :zip do
  system 'zip', '-r', "eve-overview-v#{File.read('VERSION').strip}.zip",
         build, 'README.md', 'LICENSE.txt', 'CHANGELOG.md', 'VERSION'
end

def compile_overview(name, path, overview)
  overview_hash = {}
  overview.each do |k, v|
    next if k.to_sym == :tabs
    opts = YAML.load_file(File.join(k.pluralize, "#{v}.yml"))
    if k.to_sym == :tab
      overview_hash.merge! format_tabs(opts)
    else
      overview_hash.merge! opts
    end
  end

  presets = []
  Dir['presets/*.yml'].each do |preset|
    presets << format_preset(load_preset(preset))
  end
  overview_hash.merge! 'presets' => presets

  string = "# This EVE overview generated by the EVE Online Overview Generator.\n" \
           "# https://github.com/razor-x/eve-overview\n" \
           + overview_hash.to_yaml
  File.open(path, 'w') { |f| f.write string }
end

def format_tabs(tabs)
  tab_array = []
  tabs.each_with_index do |v, i|
    v = ActiveSupport::HashWithIndifferentAccess.new v
    tab = [i, [
      ['name', format_tab_name(v)],
      ['overview', format_preset_name(load_preset_by_name(v[:overview]))],
      ['bracket', format_preset_name(load_preset_by_name(v[:bracket]))]
    ]]
    tab_array << tab
  end

  {'tabSetup' => tab_array}
end

def load_preset_by_name(preset)
  ActiveSupport::HashWithIndifferentAccess.new(
    YAML.load_file File.join('presets', "#{preset}.yml")
  )
end

def load_preset(preset)
  ActiveSupport::HashWithIndifferentAccess.new(
    YAML.load_file preset
  )
end

def format_preset(preset)
  states = merge_states(preset[:states])
  [format_preset_name(preset), [
    ['alwaysShownStates', states[:show]],
    ['filteredStates', states[:hide]],
    ['groups', merge_groups(preset[:groups])]
  ]]
end

def format_preset_name(preset)
  return "#{preset[:symbol]} #{'- ' * 4}" if preset[:level] == 0 && preset[:name].empty?
  return "#{preset[:symbol]} #{'- ' * 4}#{preset[:name]}" if preset[:level] == 0

  "#{preset[:symbol]}#{' - ' * (4 - preset[:level])}#{preset[:name]}"
end

def format_tab_name(tab)
  "<color=0x#{tab[:color]}>   #{tab[:name]}   </color>"
end

def merge_states(names)
  array = []
  states = {show: [], hide: []}
  names.each do |name|
    array << ActiveSupport::HashWithIndifferentAccess.new(
      YAML.load_file(File.join('states', "#{name}.yml"))
    )
  end
  array.each do |v|
    v.each { |k, v| states[k.to_sym].concat v }
  end
  states.each { |v| v.flatten.uniq }
end

def merge_groups(names)
  hash = ActiveSupport::HashWithIndifferentAccess.new include: [], types: []
  names.each do |name|
    sub_hash = ActiveSupport::HashWithIndifferentAccess.new YAML.load_file(File.join('groups', "#{name}.yml"))
    %i(types include).each do |k|
      hash[k].concat sub_hash[k] unless sub_hash[k].nil?
    end
  end

  reduce_group(hash).uniq.sort
end

def reduce_group(group)
  types = []
  types << group[:types] unless group[:types].nil?
  types << group[:include].map do |v|
    reduce_group ActiveSupport::HashWithIndifferentAccess.new(
      YAML.load_file(File.join('groups', "#{v}.yml"))
    )
  end unless group[:include].nil?
  types.flatten.uniq
end
