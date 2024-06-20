```yaml
the point of the fork is to try to get beautiful lyrics to work
```

# Spicetify-Nix

Modifies Spotify using [spicetify-cli](https://github.com/khanhas/spicetify-cli).

[spicetify-themes](https://github.com/morpheusthewhite/spicetify-themes) are
included and available.

## Usage

To use, add this flake to your home-manager configuration flake inputs, like so:

```nix
{
  # create an input called spicetify-nix, and set its url to this repository
  inputs.spicetify-nix.url = "github:the-argus/spicetify-nix";
}

```

And when writing your outputs function, make sure to accept `spicetify-nix` (or
whatever you chose to call it in the inputs) as a function input:

```nix
{
    outputs = { nixpkgs, spicetify-nix, ...}: let # notice spicetify-nix here
        pkgs = import nixpkgs { system = "x84_64-linux"; };
    in {
      homeConfigurations."your_username" = home-manager.lib.homeManagerConfiguration {
        inherit pkgs;
        extraSpecialArgs = {inherit spicetify-nix;};
        modules = [
            ./home.nix
            ./spicetify.nix # file where you configure spicetify
        ];
      };
    }
}
```

An even more general solution is available, though, which isn't necessary but
is recommended, as it will prevent you from having to manually put all other
flake inputs you may add in the future to `extraSpecialArgs`. You can make a
variable which contains all of your flake inputs, no matter what you change them
to, and just pass that to `extraSpecialArgs`. Home manager even has a reserved
argument inside of `extraSpecialArgs` for that: `inputs`.

```nix
{
    # the ... lets us accept any inputs, and "@ inputs" lets us capture those.
    outputs = { nixpkgs, ...} @ inputs: let
        # here we use nixpkgs from our inputs, which is why why included it
        # above instead of just {...} @ inputs. If we did that, this would be
        # "inputs.nixpkgs".
        pkgs = import nixpkgs { system = "x86_64-linux"; };
    in {
      homeConfigurations."your_username" = home-manager.lib.homeManagerConfiguration {
        inherit pkgs;
        # put our flake inputs into the "inputs" argument of extraSpecialArgs.
        extraSpecialArgs = {inherit inputs;};
        modules = [
            ./home.nix
            ./spicetify.nix # file where you configure spicetify
        ];
      };
    }
}
```

For more information on the (many) different ways of passing flake inputs to
modules can be found in [this wonderful blog post by Nobbz](https://blog.nobbz.dev/2022-12-12-getting-inputs-to-modules-in-a-flake/)

If you want to do it as a NixOS module instead of a home-manager module, the
process is the same except you use `specialArgs` instead of `extraSpecialArgs`
in `nixpkgs.lib.nixosSystem`. Also be sure to import `spicetify-nix.nixosModule`
instead of `spicetify-nix.homeManagerModule`, which you'll see happen in then next
section.

## Configuration examples

Here are two examples of files which configure spicetify when imported into a
user's home-manager configuration.

### Minimal Configuration

```nix
{ pkgs, lib, spicetify-nix, ... }:
let
  spicePkgs = spicetify-nix.packages.${pkgs.system}.default;
in
{
  # allow spotify to be installed if you don't have unfree enabled already
  nixpkgs.config.allowUnfreePredicate = pkg: builtins.elem (lib.getName pkg) [
    "spotify"
  ];

  # import the flake's module for your system
  imports = [ spicetify-nix.homeManagerModule ];

  # configure spicetify :)
  programs.spicetify =
    {
      enable = true;
      theme = spicePkgs.themes.catppuccin;
      colorScheme = "mocha";

      enabledExtensions = with spicePkgs.extensions; [
        fullAppDisplay
        shuffle # shuffle+ (special characters are sanitized out of ext names)
        hidePodcasts
      ];
    };
}
```

### Maximal configuration

NOTE: the purpose of this configuration is to demonstrate all the possible
customization options. It's not sane at all, and I see no reason why you
should actually use this one.

```nix
{
  pkgs,
  # this is the same as pkgs but the url is github:nixos/nixpkgs-unstable
  unstable,
  # this is pkgs.lib
  lib,
  # this is the input to the flake with url github:the-argus/spicetify-nix
  spicetify-nix,
  ...
}:
let
  spicePkgs = spicetify-nix.packages.${pkgs.system}.default;
in
{
  # allow spotify to be installed if you don't have unfree enabled already
  nixpkgs.config.allowUnfreePredicate = pkg: builtins.elem (lib.getName pkg) [
    "spotify"
  ];

  # import the flake's module
  imports = [ spicetify-nix.homeManagerModules.default ];

  # configure spicetify :)
  programs.spicetify =
    let
      # use a different version of spicetify-themes than the one provided by
      # spicetify-nix
      officialThemesOLD = pkgs.fetchgit {
        url = "https://github.com/spicetify/spicetify-themes";
        rev = "c2751b48ff9693867193fe65695a585e3c2e2133";
        sha256 = "0rbqaxvyfz2vvv3iqik5rpsa3aics5a7232167rmyvv54m475agk";
      };
      # pin a certain version of the localFiles custom app
      localFilesSrc = pkgs.fetchgit {
        url = "https://github.com/hroland/spicetify-show-local-files/";
        rev = "1bfd2fc80385b21ed6dd207b00a371065e53042e";
        sha256 = "01gy16b69glqcalz1wm8kr5wsh94i419qx4nfmsavm4rcvcr3qlx";
      };
    in
    {
      # use spotify from the nixpkgs master branch
      spotifyPackage = unstable.spotify;

      # use a custom build of spicetify
      spicetifyPackage = pkgs.spicetify-cli.overrideAttrs (oa: rec {
        pname = "spicetify-cli";
        version = "2.14.1";
        src = pkgs.fetchgit {
          url = "https://github.com/spicetify/${pname}";
          rev = "v${version}";
          sha256 = "sha256-262tnSKX6M9ggm4JIs0pANeq2JSNYzKkTN8awpqLyMM=";
        };
        vendorSha256 = "sha256-E2Q+mXojMb8E0zSnaCOl9xp5QLeYcuTXjhcp3Hc8gH4=";
      });

      # actually enable the installation of spotify and spicetify
      enable = true;

      # custom Dribbblish theme
      theme = {
        name = "Dribbblish";
        src = officialThemesOLD;
        requiredExtensions = [
          # define extensions that will be installed with this theme
          {
            # extension is "${src}/Dribbblish/dribbblish.js"
            filename = "dribbblish.js";
            src = "${officialThemesOLD}/Dribbblish";
          }
        ];
        appendName = true; # theme is located at "${src}/Dribbblish" not just "${src}"

        # changes to make to config-xpui.ini for this theme:
        patches = {
          "xpui.js_find_8008" = ",(\\w+=)32,";
          "xpui.js_repl_8008" = ",$\{1}56,";
        };
        injectCss = true;
        replaceColors = true;
        overwriteAssets = true;
        sidebarConfig = true;
      };

      # specify that we want to use our custom colorscheme
      colorScheme = "custom";

      # color definition for custom color scheme. (rosepine)
      customColorScheme = {
        text = "ebbcba";
        subtext = "F0F0F0";
        sidebar-text = "e0def4";
        main = "191724";
        sidebar = "2a2837";
        player = "191724";
        card = "191724";
        shadow = "1f1d2e";
        selected-row = "797979";
        button = "31748f";
        button-active = "31748f";
        button-disabled = "555169";
        tab-active = "ebbcba";
        notification = "1db954";
        notification-error = "eb6f92";
        misc = "6e6a86";
      };

      enabledCustomApps = with spicePkgs.apps; [
        new-releases
        {
          name = "localFiles";
          src = localFilesSrc;
          appendName = false;
        }
      ];
      enabledExtensions = with spicePkgs.extensions; [
        playlistIcons
        lastfm
        genre
        historyShortcut
        hidePodcasts
        fullAppDisplay
        shuffle
      ];
    };
}
```

## Themes, Extensions, and CustomApps

Are found in [THEMES.md](./THEMES.md), [EXTENSIONS.md](./EXTENSIONS.md), and
[CUSTOMAPPS.md](./CUSTOMAPPS.md), respectively.

## macOS

This package has no macOS support, because Spotify in nixpkgs has no macOS support.
