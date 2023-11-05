# Réduire la taille d'un binaire Rust

[![GitHub Actions][github-actions-badge]](https://github.com/adriantombu/min-sized-rust/actions)

[github-actions-badge]: https://github.com/adriantombu/min-sized-rust/workflows/build/badge.svg

Ce dépôt démontre comment minimiser la taille d'un binaire Rust.

Par défaut, Rust optimise la vitesse d'exécution, la vitesse de compilation et la facilité de débogage plutôt que la taille des binaires, car pour la grande majorité des applications, c'est l'idéal. Mais pour les situations où un développeur souhaite vraiment optimiser pour la taille du binaire, Rust fournit des mécanismes pour y parvenir.

# Compiler en mode release

![Version Rust Minimale : 1.0](https://img.shields.io/badge/Version%20Rust%20Minimale-1.0-brightgreen.svg)

Par défaut, `cargo build` compile le binaire Rust en mode debug. Le mode débogage désactive de nombreuses optimisations, ce qui aide les debuggers (et les IDE qui les exécutent) à fournir une meilleure expérience de débogage. Les binaires de débogage peuvent être plus lourds de 30% ou plus que les binaires de release.

Pour minimiser la taille d'un binaire, compilez en mode release ::

```bash
$ cargo build --release
```

# Retirer les symboles du binaires avec `strip`

![OS: *nix](https://img.shields.io/badge/OS-*nix-brightgreen.svg)
![Version Rust Minimale : 1.59](https://img.shields.io/badge/Version%20Rust%20Minimale-1.59-brightgreen.svg)

Par défaut, sous Linux et macOS, des informations sur les symboles sont incluses dans le fichier `.elf` compilé. Ces informations ne sont pas nécessaires pour exécuter correctement le binaire.

Cargo peut être configuré pour [`strip` automatiquement les binaires](https://doc.rust-lang.org/cargo/reference/profiles.html#strip). Il suffit de modifier `Cargo.toml` de cette manière:

```toml
[profile.release]
strip = true  # Supprime automatiquement les symboles du binaire.
```

**Avant Rust 1.59**, il faut manuellement lancer la commande [`strip`](https://linux.die.net/man/1/strip) sur le fichier `.elf` à la place:

```bash
$ strip target/release/min-sized-rust
```

# Optimiser la taille

![Version Rust Minimale : 1.28](https://img.shields.io/badge/Version%20Rust%20Minimale-1.28-brightgreen.svg)

[Par défaut, le niveau d'optimisation de Cargo est de `3` pour les versions de développement.][cargo-profile], qui optimise le binaire pour la **vitesse**. Pour demander à Cargo d'optimiser pour une **taille binaire** minimale, utilisez le niveau d'optimisation `z` dans [`Cargo.toml`](https://doc.rust-lang.org/cargo/reference/manifest.html):

[cargo-profile]: https://doc.rust-lang.org/cargo/reference/profiles.html#default-profiles

```toml
[profile.release]
opt-level = "z"  # Optimiser la taille
```

Notez que dans certains cas, le niveau `"s"` peut donner lieu à un binaire plus petit que le niveau `"z"`, comme expliqué dans la [documentation `opt-level`](https://doc.rust-lang.org/cargo/reference/profiles.html#opt-level):

> Il est recommandé d'expérimenter avec différents niveaux pour trouver le bon équilibre pour votre projet. Il peut y avoir des résultats surprenants, tels que ... les niveaux `"s"` et `"z"` ne sont pas nécessairement plus petits.

# Activer le Link Time Optimization (LTO)

![Version Rust Minimale : 1.0](https://img.shields.io/badge/Version%20Rust%20Minimale-1.0-brightgreen.svg)

Par défault, [Cargo demande aux unités de compilation d'être compilées et optimisées de manière isolée][cargo-profile]. [LTO](https://llvm.org/docs/LinkTimeOptimization.html) demande au linker d'otpimiser à l'étape du link. Cela permet, par exemple, de supprimer le code mort et souvent de réduire la taille des binaires.

Activer LTO dans `Cargo.toml`:

```toml
[profile.release]
lto = true
```

# Supprimer Jemalloc

![Version Rust Minimale : 1.28](https://img.shields.io/badge/Version%20Rust%20Minimale-1.28-brightgreen.svg)
![Version Rust Maximale : 1.31](https://img.shields.io/badge/Version%20Rust%20Maximale-1.31-brightgreen.svg)

A partir de Rust 1.32, [`jemalloc` est supprimé par défault](https://blog.rust-lang.org/2019/01/17/Rust-1.32.0.html). **Si vous utilisez Rust 1.32 ou plus récent, aucune action n'est nécessaire pour réduire la taille des binaires en ce qui concerne cette fonctionnalité**.

**Avant Rust 1.32**, pour améliorer les performances sur certaines plateformes, Rust a intégré [jemalloc](https://github.com/jemalloc/jemalloc), un allocateur souvent plus performant que l'allocateur par défaut du système. L'intégration de jemalloc a néanmoins rajouté environ 200KB à la taille finale du binaire.

Pour supprimer `jemalloc` sur Rust 1.28 - Rust 1.31, ajoutez ce code en haut de votre fichier `main.rs`:

```rust
use std::alloc::System;

#[global_allocator]
static A: System = System;
```

# Réduire l'exécution parallèle d'unités de génération de code pour améliorer l'optimisation

[Par défaut][cargo-profile], Cargo utilise 16 unités de génération en parallèle pour les versions de production. Cela améliore les temps de compilation, mais empêche certaines optimisations.

Fixez cette valeur à `1` dans `Cargo.toml` pour permettre une optimisation maximale de la taille du binaire :

```toml
[profile.release]
codegen-units = 1
```

# Interrompre en cas de panic

![Version Rust Minimale : 1.10](https://img.shields.io/badge/Version%20Rust%20Minimale-1.10-brightgreen.svg)

> **Note**: Jusqu'à présent, les options discutées pour réduire la taille des binaires n'ont pas eu d'impact sur le comportement du programme (seulement sur sa vitesse d'exécution). Cette fonctionnalité a un impact sur le comportement.

[Par défaut][cargo-profile], lorsque Rust rencontre une situation où il doit appeler `panic!()`, il déroule la pile et produit un message utile. Cela requiert néanmoins du code en plus qui augmente la taille du binaire. Il est possible de configurer `rustc` pour interrompre le programme immédiatement, ce qui enlève ce besoin de code supplémentaire.

Activez cela dans `Cargo.toml`:

```toml
[profile.release]
panic = "abort"
```

# Retirer les détails d'emplacement

![Version Rust Minimale : Nightly](https://img.shields.io/badge/Version%20Rust%20Minimale-nightly-orange.svg)

Par défaut, Rust inclut les informations de fichier, ligne et colonne dans `panic!()` et `[track_caller]` pour donner des informations utiles. Ces informations occupent de l'espace dans le fichier binaire et augmentent donc la taille des fichiers compilés.

Pour supprimer ces informations de fichier, ligne et colonne, utilisez le flag instable [`rustc` `-Zlocation-detail`](https://github.com/rust-lang/rfcs/blob/master/text/2091-inline-semantic.md#location-detail-control) :

```bash
$ RUSTFLAGS="-Zlocation-detail=none" cargo +nightly build --release
```

# Optimiser `libstd` avec `build-std`

![Version Rust Minimale : Nightly](https://img.shields.io/badge/Version%20Rust%20Minimale-nightly-orange.svg)

> **Note**: Voir également [Xargo](https://github.com/japaric/xargo), qui est le prédécesseur de `build-std`. [Xargo est actuellement en mode maintenance](https://github.com/japaric/xargo/issues/193).

> Un exemple de projet est disponible de le dossier [`build_std`](build_std).

Rust fournit des copies précompilée de la bibliothèque standard (`libstd`) dans sa suite d'outils. Cela signifie que les développeurs n'ont pas besoin de compiler `libstd` à chaque fois qu'ils créent leurs applications. `libstd` est lié statiquement dans le binaire à la place.

Bien que cela soit très pratique, il y a plusieurs inconvénients si un développeur essaie d'optimiser la taille de manière agressive.

1. La `libstd` précompilée est optimisée pour la vitesse, pas pour la taille.

2. Il n'est pas possible de retirer des portions de `libstd`  qui ne sont pas utilisé dans une application en particulier (par exemple LTO et le comportement de panic).

C'est ici que [`build-std`](https://doc.rust-lang.org/cargo/reference/unstable.html#build-std) se retrouve utile. La feature `build-std` est capable de compiler `libstd` dans votre application depuis la source. Il le fait avec le composant `rust-src` que `rustup` fournit commodément.

Installez la suite d'outils appropriée et le composant `rust-src` :

```bash
$ rustup toolchain install nightly
$ rustup component add rust-src --toolchain nightly
```

Compiler en utilisant `build-std`:

```bash
# Afficher le triplet de votre machine 
$ rustc -vV
...
host: x86_64-apple-darwin

# Utilisez ce triplet quand vous compilez avec build-std.
# Ajoutez l'option =std,panic_abort pour permettre à l'option panic = "abort" de Cargo.toml de fonctionner.
# Voir: https://github.com/rust-lang/wg-cargo-std-aware/issues/56
$ RUSTFLAGS="-Zlocation-detail=none" cargo +nightly build -Z build-std=std,panic_abort --target x86_64-apple-darwin --release
```

Sur macOS, la taille finale du binaire est réduite à 51 Ko.

# Supprimer le formatting des chaînes de caractères de `panic` avec `panic_immediate_abort`

![Version Rust Minimale : Nightly](https://img.shields.io/badge/Version%20Rust%20Minimale-nightly-orange.svg)

Même si `panic = "abort"` est spécifié dans `Cargo.toml`, `rustc` il inclut toujours par défaut les chaînes de caractère et le code de formatage du panic dans le binaire final. [Une feature instable `panic_immediate_abort`](https://github.com/rust-lang/rust/pull/55011) a été mergée dans le compileur `nightly` `rustc` pour addresser cela.

Pour l'utiliser, répétez les instructions au dessus pour utiliser `build-std`, mais passez également l'option suivante [`-Z build-std-features=panic_immediate_abort`](https://doc.rust-lang.org/cargo/reference/unstable.html#build-std-features).

```bash
$ cargo +nightly build -Z build-std=std,panic_abort -Z build-std-features=panic_immediate_abort \
    --target x86_64-apple-darwin --release
```

Sur macOS, la taille finale du binaire est réduite à 30 Ko.

# Supprimer `core::fmt` avec `#![no_main]` et un usage prudent de `libstd`

![Version Rust Minimale : Nightly](https://img.shields.io/badge/Version%20Rust%20Minimale-nightly-orange.svg)

> Un exemple de projet est disponible de le dossier [`no_main`](no_main).

> Cette section est en partie possible grâce à [@vi](https://github.com/vi)
 
Jusqu'à présent, nous n'avons pas restreint les utilitaires de `libstd` que nous utilisions. Dans cette section, nous allons restreindre notre utilisation de `libstd` afin de réduire encore plus la taille des binaires.

Si vous voulez un exécutable de moins de 20 kilo-octets, le code de formatage des chaînes de Rust, [`core::fmt`](https://doc.rust-lang.org/core/fmt/index.html) doit être supprimé. `panic_immediate_abort` ne supprime que certaines utilisations de ce code. Il y a beaucoup d'autres codes qui utilisent le formatage dans certains cas. Cela inclut le code "pre-main" de Rust dans `libstd`.

En utilisant un point d'entrée C (en ajoutant l'attribut `# ![no_main]`), en gérant stdio manuellement, et en analysant soigneusement les morceaux de code que vous ou vos dépendances incluez, vous pouvez parfois utiliser `libstd` tout en évitant `core::fmt`.

Il faut s'attendre à ce que le code soit plus complexe et non portable, avec plus de `unsafe{}` que d'habitude. Cela ressemble à `no_std`, mais avec `libstd`.

Commencez avec un exécutable vide, assurez-vous que [`xargo bloat --release --target=...`](https://github.com/RazrFalcon/cargo-bloat) ne contient pas `core::fmt` ou quelque chose relatif au padding. Ajoutez (ou décommentez) un peu de code et confirmez que `xargo bloat` relève beaucoup plus d'information désormais. Examinez le code source que vous venez d'ajouter. Il est probable qu'une librairie externe ou une nouvelle fonction `libstd` est utilisée. Tenez-en compte dans votre processus d'évaluation (cela vous force à utiliser la section `[replace]` et peut-être chercher dans `libstd`) et essayez de comprendre pourquoi cela prend plus de place que ça ne devrait. Choisisez une alternative ou patchez les dépendances pour éviter des features non nécessaires. Décommentez un peu plus le code, et continuez à débugger avec `xargo bloat`.

Sur macOS, la taille finale du binaire est réduite à 8 Ko.

# Supprimer `libstd` avec `#![no_std]`

![Version Rust Minimale : 1.30](https://img.shields.io/badge/Version%20Rust%20Minimale-1.30-brightgreen.svg)

> Un exemple de projet est disponible de le dossier [`no_std`](no_std).

Jusqu'à présent, notre application utilisait la bibliothèque standard Rust, `libstd`. `libstd` fournit de nombreuses APIs cross-plateforme pratique et bien testées. Mais si un utilisateur veut réduire la taille d'un programme binaire à une taille équivalente à celle d'un programme C, il est possible de ne dépendre que de `libc`.

Il est important de comprendre que cette approche présente de nombreux inconvénients. Par exemple, vous devrez probablement écrire beaucoup de code `unsafe` et perdre l'accès à la majorité des crates Rust qui dépendent de `libstd`. Néanmoins, il s'agit d'une option (bien qu'extrême) pour réduire la taille des binaires.

Un binaire réduit de cette manière est autour de 8 Ko.

```rust
#![no_std]
#![no_main]

extern crate libc;

#[no_mangle]
pub extern "C" fn main(_argc: isize, _argv: *const *const u8) -> isize {
    // Puisque nous transmettons une chaîne de caractères C, le caractère nul final est obligatoire.
    const HELLO: &'static str = "Hello, world!\n\0";
    unsafe {
        libc::printf(HELLO.as_ptr() as *const _);
    }
    0
}

#[panic_handler]
fn my_panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}
```

# Compresser le binaire

Jusqu'à présent, toutes les techniques de réduction de la taille étaient spécifiques à Rust. Cette section décrit un outil de compression agnostique au language qui permet de réduire davantage la taille des binaires.

[UPX](https://github.com/upx/upx) est un outil puissant permettant de créer un fichier binaire autonome et compressé sans exigence supplémentaire en matière d'exécution. Il prétend réduire la taille des binaires de 50 à 70 %, mais le résultat réel dépend de votre exécutable.

```bash
$ upx --best --lzma target/release/min-sized-rust
```

Il convient de noter qu'il est arrivé que des binaires contenant des fichiers UPX soient détectés par des logiciels antivirus basés sur une méthode heuristique, car les logiciels malveillants utilisent souvent UPX.

# Outils

- [`cargo-bloat`](https://github.com/RazrFalcon/cargo-bloat) - Déterminez ce qui occupe le plus d'espace dans votre exécutable.
- [`cargo-unused-features`](https://github.com/TimonPost/cargo-unused-features) - Trouvez et éliminez de votre projet les features activées mais potentiellement inutilisées.
- [`momo`](https://github.com/llogiq/momo) - `proc_macro`  librairie utilisée pour limiter l'empreinte des méthodes génériques dans le code.
- [Twiggy](https://rustwasm.github.io/twiggy/index.html) - Un profileur de taille de code pour Wasm.

# Conteneurs

Il est parfois avantageux de déployer Rust dans un container (par exemple [Docker](https://www.docker.com/)). Il existe plusieurs ressources intéressantes pour aider à créer des images de conteneurs de taille minimale qui exécutent les binaires Rust.

- [L'image `rust:alpine` officielle](https://hub.docker.com/_/rust)
- [mini-docker-rust](https://github.com/kpcyrd/mini-docker-rust)
- [muslrust](https://github.com/clux/muslrust)
- [docker-slim](https://github.com/docker-slim/docker-slim) - Minifier les images Docker

# Références

- [151-byte static Linux binary in Rust - 2015][151-byte-static-linux-binary]
- [Why is a Rust executable large? - 2016][why-rust-binary-large]
- [Tiny Rocket - 2018](https://jamesmunns.com/blog/tinyrocket/)
- [Formatting is Unreasonably Expensive for Embedded Rust - 2019][fmt-unreasonably-expensive]
- [Tiny Windows executable in Rust - 2019][tiny-windows-exe]
- [Making a really tiny WebAssembly graphics demos - 2019][tiny-webassembly-graphics]
- [Reducing the size of the Rust GStreamer plugin - 2020][gstreamer-plugin]
- [Optimizing Rust Binary Size - 2020][optimizing-rust-binary-size]
- [Minimizing Mender-Rust - 2020][minimizing-mender-rust]
- [Optimize Rust binaries size with cargo and Semver - 2021][optimize-with-cargo-and-semver]
- [Tighten rust’s belt: shrinking embedded Rust binaries - 2022][tighten-rusts-belt]
- [Avoiding allocations in Rust to shrink Wasm modules - 2022][avoiding-allocations-shrink-wasm]
- [A very small Rust binary indeed - 2022][a-very-small-rust-binary]
- [`min-sized-rust-windows`][min-sized-rust-windows] - Astuces spécifiques à Windows pour réduire la taille des binaires
- [Shrinking `.wasm` Code Size][shrinking-wasm-code-size]

[151-byte-static-linux-binary]: https://mainisusuallyafunction.blogspot.com/2015/01/151-byte-static-linux-binary-in-rust.html
[why-rust-binary-large]: https://lifthrasiir.github.io/rustlog/why-is-a-rust-executable-large.html
[fmt-unreasonably-expensive]: https://jamesmunns.com/blog/fmt-unreasonably-expensive/
[tiny-windows-exe]: https://www.codeslow.com/2019/12/tiny-windows-executable-in-rust.html
[tiny-webassembly-graphics]: https://cliffle.com/blog/bare-metal-wasm/
[gstreamer-plugin]: https://www.collabora.com/news-and-blog/blog/2020/04/28/reducing-size-rust-gstreamer-plugin/
[optimizing-rust-binary-size]: https://arusahni.net/blog/2020/03/optimizing-rust-binary-size.html
[minimizing-mender-rust]: https://mender.io/blog/building-mender-rust-in-yocto-and-minimizing-the-binary-size
[optimize-with-cargo-and-semver]: https://oknozor.github.io/blog/optimize-rust-binary-size/
[tighten-rusts-belt]: https://dl.acm.org/doi/abs/10.1145/3519941.3535075
[avoiding-allocations-shrink-wasm]: https://nickb.dev/blog/avoiding-allocations-in-rust-to-shrink-wasm-modules/
[a-very-small-rust-binary]: https://darkcoding.net/software/a-very-small-rust-binary-indeed/
[min-sized-rust-windows]: https://github.com/mcountryman/min-sized-rust-windows
[shrinking-wasm-code-size]: https://rustwasm.github.io/docs/book/reference/code-size.html
