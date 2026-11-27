# Proposal: native Graphics settings for the existing FSR 4K presets

Add one opt-in **Settings → Graphics → 4K Upscaler** row using the native menu prototype from [Marvisak/VIctory#2863](https://github.com/Marvisak/VIctory/pull/2863) and wire it into your existing FSR presets.

| Native choice | What the fork uses on the next launch |
|---|---|
| 🎮 **Title default** | The title's default renderer |
| 🖼️ **FSR Quality** | 2560 × 1440 → 3840 × 2160 |
| ⚖️ **FSR Balanced** | 2258 × 1270 → 3840 × 2160 |
| ⚡ **FSR Performance** | 1920 × 1080 → 3840 × 2160 |

The upstream prototype has exercised native navigation, custom option labels, and settings persistence in game. This draft proposes the fork-specific wiring and a test plan. For the first pass I'd verify the row with a controller, check the saved value after reopening Settings, then restart and confirm the render sizes in the existing dispatch log.

## Rust example

A typed enum declares the labels, menu order, default, and TOML values. Adding another row uses the same `Choice` trait.

```rust
use crate::settings::{Choice, ConfigKey, Opt, Text};

#[derive(Debug, Default, Clone, Copy, PartialEq, Eq, serde::Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Fsr4kMode {
    #[default]
    TitleDefault,
    Quality,
    Balanced,
    Performance,
}

impl Choice for Fsr4kMode {
    const CONFIG: ConfigKey = ConfigKey { section: "graphics", key: "fsr_4k" };
    const NAME: Text = Text::own("victory_fsr_4k", "4K Upscaler");
    const HELP: Text = Text::own("victory_fsr_4k_help", "Applies on the next launch.");
    const OPTIONS: &'static [Opt<Self>] = &[
        Opt { value: Self::TitleDefault, label: Text::own("victory_fsr_4k_default", "Title default"), toml: "title_default" },
        Opt { value: Self::Quality, label: Text::own("victory_fsr_4k_quality", "FSR Quality"), toml: "quality" },
        Opt { value: Self::Balanced, label: Text::own("victory_fsr_4k_balanced", "FSR Balanced"), toml: "balanced" },
        Opt { value: Self::Performance, label: Text::own("victory_fsr_4k_performance", "FSR Performance"), toml: "performance" },
    ];

    fn from_config(config: &crate::Config) -> Self {
        config.fsr_4k_mode
    }
}
```

The proposed port loads the new `Config::fsr_4k_mode` field from `[graphics].fsr_4k` and registers the row with `ListSetting::of::<Fsr4kMode>()`. At startup, pass the selected preset to the existing `prepare_fsr_4k` path. The existing `-Fsr4K` launcher option and `VICTORY_FSR_4K=0` keep their override behavior.

```rust
fn requested(config: &Config) -> anyhow::Result<Option<FsrPreset>> {
    if std::env::var_os("VICTORY_FSR_4K").is_some() {
        return FsrPreset::requested();
    }
    Ok(match config.fsr_4k_mode {
        Fsr4kMode::TitleDefault => None,
        Fsr4kMode::Quality => Some(FsrPreset::Quality),
        Fsr4kMode::Balanced => Some(FsrPreset::Balanced),
        Fsr4kMode::Performance => Some(FsrPreset::Performance),
    })
}
```

The existing retail-build checks and `[paths].fidelityfx_dir` SDK setup still apply.

[Upstream design and testing](https://github.com/Marvisak/VIctory/issues/2862) · [Fork launcher and FSR setup](https://github.com/avatarmanz/VIctory-7900XTX/blob/main/README.md#how-to-run-it)
