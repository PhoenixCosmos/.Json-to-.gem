# .Json-to-.gem
useful tool to create gem files. Put inside of folder with json file, edit json_data = JSON.parse(File.read('gem_01.json')) with the json file name. Run the script: ruby build_plugin.rb. Navigate into the folder. Rebuild the gem: rake package
