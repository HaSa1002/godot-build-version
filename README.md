# Godot Build Version

This addon calculates version info for your project that can be displayed inside it. The version info is saved in the project settings while avoiding SCM noise.

**NOTE: You absolutely need a git version controlled repository to make this addon work!** 

## Usage

Extract the addons folder into your project and activate the plugin in the project settings.

The addon adds five new project settings into your `project.godot`:

```
application/version/dirty=false
application/version/broken=false
application/version/custom=false
application/version/commit=""
application/version/label=""
```

The first four will always be reset to the default once the export is done or you stopped playing to prevent SCM noise. `application/value/label` can be overwritten to a default/fallback value of your liking.

Assuming you have a `Version` `Label`, you can display the version like this:

```gdscript
func _ready() -> void:
	$Version.text = "Version: %s" % ProjectSettings["application/version/label"]
	
	# Let's also add some colour coding to easily recognise specific states
	if ProjectSettings["application/version/broken"]:
		$Version.add_color_override("font_color", Color.webmaroon)
	elif ProjectSettings["application/version/dirty"]:
		$Version.add_color_override("font_color", Color.yellow)
	elif ProjectSettings["application/version/custom"]:
		$Version.add_color_override("font_color", Color.blue)
	else:
		$Version.add_color_override("font_color", Color.green)
```

## Project setting saving notes

Because Godot saves *everything* when you use `ProjectSettings.save_custom()` you need to be extra careful as the version data **will** get written into `user://override.cfg`. You'll need to filter out most of the sections. In Libre Train Sim we purge our settings like this:

```gdscript
func save_settings():
	# Save, except if in Godot Editor
	if OS.has_feature("editor"):
		return
	var err := ProjectSettings.save_custom("user://override.cfg")
	if err != OK:
		Logger.err("Failed to save user settings", self)
		# Nothing was saved, so nothing needs special treatment
		return
	# _global_script_classes gets saved, too
	# ...
	# and overriden on load ...
	# Meaning, any new class that we register in an update will not work causing a crash
	# this IS a Godot bug, but until I have time to fix it, let's just work around
	# https://github.com/godotengine/godot/issues/61556 may be intersting as well
	var file := ConfigFile.new()
	err = file.load("user://override.cfg")
	if err != OK:
		Logger.err("Couldn't load user://override.cfg for _global_script_classes" + \
				"stripping. This is a major functionality threat! " + \
				"Please ensure removal of the file!", self)
		return
	file.erase_section_key("", "_global_script_classes")
	file.erase_section_key("", "_global_script_class_icons")

	# We forcefully delete a good chunk of things that may get into our way
	file.erase_section("network")
	file.erase_section("editor_plugins")
	file.erase_section("editor")
	file.erase_section("autoload")
	file.erase_section("audio")
	file.erase_section("application") # Also removes the version entries

	err = file.save("user://override.cfg")
	if err != OK:
		Logger.err("Couldn't load user://override.cfg for _global_script_classes" + \
				"stripping. This is a major functionality threat! " + \
				"Please ensure removal of the file!", self)
```

## Calculation of the version label

The version is based on the tag. The tag is the base. If there's no tag, the commit hash will be shown. So to get started tag some commit with something like `v0.1.dev`. It doesn't really matter as it is copied 1:1 as the first part into the version label.

The build is tagged as `custom` if it is *not* on `main` or `master`, or a branch name that includes `release`. A custom build gets the commit hash appended.

If the commit is tagged, then the build number will be ommited.

Otherwise the label looks like `<TAG>.<BUILD>` and depending on the state `.custom.<COMMIT> [BROKEN] [DIRTY BRANCH]`.

A final label in a pull request with uncommited local changes can look like this: `v0.9.beta1.190.custom.e6a4d32 DIRTY BRANCH`. `v0.9.beta1` was the last tag 190 commits ago.

The data is gathered by asking git three questions:

```
git describe --all
git status -s
git describe --tags --long --always --dirty= --broken=?
```

For `v0.9.beta1.190.custom.e6a4d32 DIRTY BRANCH` the output looks like this:

```
> git describe --all
heads/add/version-info
> git status -s     
 M src/addons/jean28518.jTools/jSettings/input_map_resource.gd
 M src/addons/jean28518.jTools/jSettings/input_presets/layout_lts.tres
 M src/addons/jean28518.jTools/jSettings/input_presets/layout_openbve.tres
 M src/project.godot
> git describe --tags --long --always --dirty= --broken=?
v0.9.beta1-190-ge6a4d32
```

Feel free to query for more and/or different formats and more granular data. The addon was originally developed for Libre Train Sim. The intention for this version string is to allow easy distribution of development builds while staying able to pin point bug reports to the specific commit and version used.

## Additional info

There's [GodotVersion](https://github.com/Gregorein/GodotVersion) if you need semantic versioning.