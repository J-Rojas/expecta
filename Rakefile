require 'tmpdir'
require 'pathname'

PROJECT = 'Expecta.xcodeproj'
CONFIGURATION = 'Release'

NO_COLOR= "\033[0m"
GREEN_COLOR = "\033[32;01m"

def execute(command, stdout=nil)
  puts "Running #{command}..."
  command += " > #{stdout}" if stdout
  system(command) or raise "** BUILD FAILED **"
end

def test(scheme)
  execute "xcodebuild -project #{PROJECT} -scheme #{scheme} -configuration #{CONFIGURATION} test SYMROOT=build | xcpretty -c && exit ${PIPESTATUS[0]}"
end

def build(scheme, sdk, product)
  execute "xcodebuild -project #{PROJECT} -scheme #{scheme} -sdk #{sdk} -configuration #{CONFIGURATION} SYMROOT=build"
  build_dir = "#{CONFIGURATION}#{sdk == 'macosx' ? '' : "-#{sdk}"}"
  "Expecta/build/#{build_dir}/#{product}"
end

def build_framework(scheme, sdk)
  build(scheme, sdk, 'Expecta.framework')
end

def build_static_lib(scheme, sdk)
  build(scheme, sdk, 'libExpecta.a')
end

def lipo(bin1, bin2, output)
  execute "lipo -create '#{bin1}' '#{bin2}' -output '#{output}'"
end

def clean(scheme)
  execute "xcodebuild -project #{PROJECT} -scheme #{scheme} clean"
end

def puts_green(str)
  puts "#{GREEN_COLOR}#{str}#{NO_COLOR}"
end

desc 'Run tests'
task :test do |t|
  execute "xcodebuild test -project #{PROJECT} -scheme Expecta"
end

desc 'clean'
task :clean do |t|
  puts_green '=== CLEAN ==='
  clean('Expecta')
  clean('Expecta-iOS')
end

desc 'build'
task :build => :clean do |t|
  puts_green "=== BUILD ==="

  osx_framework     = build_framework('Expecta', 'macosx')
  ios_sim_framework = build_framework('Expecta-iOS', 'iphonesimulator')
  ios_framework     = build_framework('Expecta-iOS', 'iphoneos')

  osx_static_lib     = build_static_lib('libExpecta', 'macosx')
  ios_sim_static_lib = build_static_lib('libExpecta-iOS', 'iphonesimulator')
  ios_static_lib     = build_static_lib('libExpecta-iOS', 'iphoneos')

  osx_build_path = Pathname.new(osx_framework).parent.to_s
  ios_build_path = Pathname.new(ios_framework).parent.to_s
  ios_univ_build_path = "Expecta/build/#{CONFIGURATION}-ios-universal"

  puts_green "\n=== GENERATE UNIVERSAL iOS BINARY (Device/Simulator) ==="
  execute "mkdir -p '#{ios_univ_build_path}'"
  execute "cp -a '#{ios_framework}' '#{ios_univ_build_path}'"
  execute "cp -a '#{ios_static_lib}' '#{ios_univ_build_path}'"

  ios_framework_name = Pathname.new(ios_framework).basename.to_s
  ios_static_lib_name = Pathname.new(ios_static_lib).basename.to_s

  ios_univ_framework  = File.join(ios_univ_build_path, ios_framework_name)
  ios_univ_static_lib = File.join(ios_univ_build_path, ios_static_lib_name)

  lipo("#{ios_framework}/Expecta", "#{ios_sim_framework}/Expecta", "#{ios_univ_framework}/Expecta")
  lipo(ios_static_lib, ios_sim_static_lib, ios_univ_static_lib)

  puts_green "\n=== CODESIGN iOS FRAMEWORK ==="
  execute "/usr/bin/codesign --force --sign 'iPhone Developer' --resource-rules='#{ios_univ_framework}'/ResourceRules.plist '#{ios_univ_framework}'"

  puts_green "\n=== COPY PRODUCTS ==="
  execute "yes | rm -rf Products"
  execute "mkdir -p Products/ios"
  execute "mkdir -p Products/osx"
  execute "cp -a #{osx_framework} Products/osx"
  execute "cp -a #{osx_static_lib} Products/osx"
  execute "cp -a #{ios_univ_framework} Products/ios"
  execute "cp -a #{ios_univ_static_lib} Products/ios"
  execute "cp -a #{osx_build_path}/usr/local/include/* Products"
  puts "\n** BUILD SUCCEEDED **"
end

namespace 'specs' do
  task :ios => :clean do |t|
    test("Expecta-iOS")
  end

  task :osx => :clean do |t|
    test("Expecta")
  end
end

task :default => [:build]
