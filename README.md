# antonio911
xenia master
/**
******************************************************************************
* Copyright 2019 Ben Vanik. All rights reserved.
 * Released under the BSD license - see LICENSE in the root for more details. *
 ******************************************************************************
  */
  
  #include <sstream>
  
  #include "xenia/kernel/kernel_state.h"
  #include "xenia/kernel/util/shim_utils.h"
  #include "xenia/kernel/xam/user_profile.h"
  
  namespace xe {
  namespace kernel {
  namespace xam {
  
UserProfile::UserProfile() {
xuid_ = 0xBABEBABEBABEBABE;
 name_ = "User";

// https://cs.rin.ru/forum/viewtopic.php?f=38&t=60668&hilit=gfwl+live&start=195
  // https://github.com/arkem/py360/blob/master/py360/constants.py
  // XPROFILE_GAMER_YAXIS_INVERSION
  AddSetting(std::make_unique<Int32Setting>(0x10040002, 0));
  // XPROFILE_OPTION_CONTROLLER_VIBRATION
  AddSetting(std::make_unique<Int32Setting>(0x10040003, 3));
  // XPROFILE_GAMERCARD_ZONE
  AddSetting(std::make_unique<Int32Setting>(0x10040004, 0));
  // XPROFILE_GAMERCARD_REGION
  AddSetting(std::make_unique<Int32Setting>(0x10040005, 0));
  // XPROFILE_GAMERCARD_CRED
  AddSetting(std::make_unique<Int32Setting>(0x10040006, 0xFA));
  // XPROFILE_GAMERCARD_REP
  AddSetting(std::make_unique<FloatSetting>(0x5004000B, 0.0f));
  // XPROFILE_OPTION_VOICE_MUTED
  AddSetting(std::make_unique<Int32Setting>(0x1004000C, 0));
  // XPROFILE_OPTION_VOICE_THRU_SPEAKERS
  AddSetting(std::make_unique<Int32Setting>(0x1004000D, 0));
  // XPROFILE_OPTION_VOICE_VOLUME
  AddSetting(std::make_unique<Int32Setting>(0x1004000E, 0x64));
  // XPROFILE_GAMERCARD_MOTTO
  AddSetting(std::make_unique<UnicodeSetting>(0x402C0011, L""));
  // XPROFILE_GAMERCARD_TITLES_PLAYED
  AddSetting(std::make_unique<Int32Setting>(0x10040012, 1));
  // XPROFILE_GAMERCARD_ACHIEVEMENTS_EARNED
  AddSetting(std::make_unique<Int32Setting>(0x10040013, 0));
  
  // XPROFILE_GAMER_DIFFICULTY
  AddSetting(std::make_unique<Int32Setting>(0x10040015, 0));
  // XPROFILE_GAMER_CONTROL_SENSITIVITY
  AddSetting(std::make_unique<Int32Setting>(0x10040018, 0));
  // Preferred color 1
  AddSetting(std::make_unique<Int32Setting>(0x1004001D, 0xFFFF0000u));
  // Preferred color 2
  AddSetting(std::make_unique<Int32Setting>(0x1004001E, 0xFF00FF00u));
  // XPROFILE_GAMER_ACTION_AUTO_AIM
  AddSetting(std::make_unique<Int32Setting>(0x10040022, 1));
  // XPROFILE_GAMER_ACTION_AUTO_CENTER
  AddSetting(std::make_unique<Int32Setting>(0x10040023, 0));
  // XPROFILE_GAMER_ACTION_MOVEMENT_CONTROL
  AddSetting(std::make_unique<Int32Setting>(0x10040024, 0));
  // XPROFILE_GAMER_RACE_TRANSMISSION
  AddSetting(std::make_unique<Int32Setting>(0x10040026, 0));
  // XPROFILE_GAMER_RACE_CAMERA_LOCATION
  AddSetting(std::make_unique<Int32Setting>(0x10040027, 0));
  // XPROFILE_GAMER_RACE_BRAKE_CONTROL
  AddSetting(std::make_unique<Int32Setting>(0x10040028, 0));
  // XPROFILE_GAMER_RACE_ACCELERATOR_CONTROL
  AddSetting(std::make_unique<Int32Setting>(0x10040029, 0));
  // XPROFILE_GAMERCARD_TITLE_CRED_EARNED
  AddSetting(std::make_unique<Int32Setting>(0x10040038, 0));
  // XPROFILE_GAMERCARD_TITLE_ACHIEVEMENTS_EARNED
  AddSetting(std::make_unique<Int32Setting>(0x10040039, 0));

// If we set this, games will try to get it.
  // XPROFILE_GAMERCARD_PICTURE_KEY
  AddSetting(
      std::make_unique<UnicodeSetting>(0x4064000F, L"gamercard_picture_key"));

// XPROFILE_TITLE_SPECIFIC1
  AddSetting(std::make_unique<BinarySetting>(0x63E83FFF));
  // XPROFILE_TITLE_SPECIFIC2
  AddSetting(std::make_unique<BinarySetting>(0x63E83FFE));
  // XPROFILE_TITLE_SPECIFIC3
  AddSetting(std::make_unique<BinarySetting>(0x63E83FFD));
}

void UserProfile::AddSetting(std::unique_ptr<Setting> setting) {
  Setting* previous_setting = setting.get();
  std::swap(settings_[setting->setting_id], previous_setting);

if (setting->is_set && setting->is_title_specific()) {
    SaveSetting(setting.get());
  }

if (previous_setting) {
    // replace: swap out the old setting from the owning list
    for (auto vec_it = setting_list_.begin(); vec_it != setting_list_.end();
         ++vec_it) {
      if (vec_it->get() == previous_setting) {
        vec_it->swap(setting);
        break;
      }
    }
  } else {
    // new setting: add to the owning list
    setting_list_.push_back(std::move(setting));
  }
}

UserProfile::Setting* UserProfile::GetSetting(uint32_t setting_id) {
  const auto& it = settings_.find(setting_id);
  if (it == settings_.end()) {
    return nullptr;
  }
  UserProfile::Setting* setting = it->second;
  if (setting->is_title_specific()) {
    // If what we have loaded in memory isn't for the title that is running
    // right now, then load it from disk.
    if (kernel_state()->title_id() != setting->loaded_title_id) {
      LoadSetting(setting);
    }
  }
  return setting;
}

void UserProfile::LoadSetting(UserProfile::Setting* setting) {
  if (setting->is_title_specific()) {
    auto content_dir =
        kernel_state()->content_manager()->ResolveGameUserContentPath();
    auto setting_id = xe::format_string(L"%.8X", setting->setting_id);
    auto file_path = xe::join_paths(content_dir, setting_id);
    auto file = xe::filesystem::OpenFile(file_path, "rb");
    if (file) {
      fseek(file, 0, SEEK_END);
      uint32_t input_file_size = static_cast<uint32_t>(ftell(file));
      fseek(file, 0, SEEK_SET);

 std::vector<uint8_t> serialized_data(input_file_size);
      fread(serialized_data.data(), 1, serialized_data.size(), file);
      fclose(file);
      setting->Deserialize(serialized_data);
      setting->loaded_title_id = kernel_state()->title_id();
    }
  } else {
    // Unsupported for now.  Other settings aren't per-game and need to be
    // stored some other way.
    XELOGW("Attempting to load unsupported profile setting from disk");
  }
}

void UserProfile::SaveSetting(UserProfile::Setting* setting) {
  if (setting->is_title_specific()) {
    auto serialized_setting = setting->Serialize();
    auto content_dir =
        kernel_state()->content_manager()->ResolveGameUserContentPath();
    xe::filesystem::CreateFolder(content_dir);
    auto setting_id = xe::format_string(L"%.8X", setting->setting_id);
    auto file_path = xe::join_paths(content_dir, setting_id);
    auto file = xe::filesystem::OpenFile(file_path, "wb");
    fwrite(serialized_setting.data(), 1, serialized_setting.size(), file);
    fclose(file);
  } else {
    // Unsupported for now.  Other settings aren't per-game and need to be
    // stored some other way.
    XELOGW("Attempting to save unsupported profile setting to disk");
  }
}

}  // namespace xam
}  // namespace kernel
}  // namespace xe
Xenia - Xbox 360 Emulator
Xenia is an experimental emulator for the Xbox 360. For more information, see the main xenia website.

Interested in supporting the core contributors? Visit Xenia Project on Patreon.

Come chat with us about emulator-related topics on Discord. For developer chat join #dev but stay on topic. Lurking is not only fine, but encouraged! Please check the frequently asked questions page before asking questions. We've got jobs/lives/etc, so don't expect instant answers.

Discussing illegal activities will get you banned. No warnings.

Status
Buildbot	Status
Windows	Build status
Linux	Build status
Quite a few real games run. Quite a few don't. See the Game compatibility list for currently tracked games, and feel free to contribute your own updates, screenshots, and information there following the existing conventions.

Disclaimer
The goal of this project is to experiment, research, and educate on the topic of emulation of modern devices and operating systems. It is not for enabling illegal activity. All information is obtained via reverse engineering of legally purchased devices and games and information made public on the internet (you'd be surprised what's indexed on Google...).

Quickstart
With Windows 8+, Python 3.4+, and Visual Studio 2017 or 2019 and the Windows SDKs installed:

> git clone https://github.com/xenia-project/xenia.git
> cd xenia
> xb setup

# Pull latest changes, rebase, and update submodules and premake:
> xb pull

# Build on command line:
> xb build

# Run premake and open Visual Studio (run the 'xenia-app' project):
> xb devenv

# Run premake to update the sln/vcproj's:
> xb premake

# Format code to the style guide:
> xb format
When fetching updates use xb pull to automatically fetch everything and run premake for project files/etc.

Building
See building.md for setup and information about the xb script. When writing code, check the style guide and be sure to run clang-format!

Contributors Wanted!
Have some spare time, know advanced C++, and want to write an emulator? Contribute! There's a ton of work that needs to be done, a lot of which is wide open greenfield fun.

For general rules and guidelines please see CONTRIBUTING.md.

Fixes and optimizations are always welcome (please!), but in addition to that there are some major work areas still untouched:

Help work through missing functionality/bugs in games
Add input drivers for PS4 controllers (or anything else)
Skilled with Linux? A strong contributor is needed to help with porting
See more projects good for contributors. It's a good idea to ask on Discord/check the issues before beginning work on something.

FAQ
For more see the main frequently asked questions page.

Can I get an exe?
Check Appveyor's artifacts to see what's there.






























