# CLAUDE.md — Guide pour agents IA

Ce fichier aide les agents IA à comprendre rapidement l'architecture du projet et à contribuer efficacement.

---

## Présentation du projet

**Papercraft** est un outil graphique desktop (Windows/Linux) pour déplier des modèles 3D en patrons imprimables (PDF, SVG, PNG). Il est écrit en **Rust** avec une interface ImGui et un rendu OpenGL.

- Fork de travail : https://github.com/creach-t/papercraft
- Original upstream : https://github.com/rodrigorc/papercraft
- Branche principale des modifications : `feature/3d-views-pdf`

---

## Environnement de développement

### Prérequis (Windows)
- **Rust** (stable, edition 2024)
- **LLVM** : `C:\Program Files\LLVM\bin` dans le PATH (pour `libclang` utilisé par `easy-imgui-sys`)
- **MSYS2 + mingw64** : `C:\msys64\mingw64\bin` dans le PATH (pour `windres.exe`, optionnel)
- Variables d'environnement utiles :
  - `LIBCLANG_PATH=C:\Program Files\LLVM\bin`
  - `WINDRES=C:\msys64\mingw64\bin\windres.exe` (optionnel, pour l'icône Windows)

### Commandes usuelles
```powershell
# Compiler
cargo build

# Lancer l'application
cargo run

# Hot-reload (recompile + relance à chaque sauvegarde)
cargo watch -x run

# Vérification rapide des erreurs (sans lancer)
cargo watch -x check

# Build release (optimisé)
cargo build --release
```

### Patch local
`Cargo.toml` contient un patch pour le crate `include-po` (bug Windows avec les backslashes) :
```toml
[patch.crates-io]
include-po = { path = "E:/Projets/papercraft-maker/include-po-patch" }
```
Le patch est dans `E:\Projets\papercraft-maker\include-po-patch\src\lib.rs` — il remplace `\\` par `/` dans le `#[path = "..."]` généré.

---

## Architecture du code

### Structure des fichiers (`src/`)

| Fichier | Rôle |
|---|---|
| `main.rs` | Point d'entrée, boucle événements, `GlobalContext`, actions fichier, exports |
| `ui.rs` | `PapercraftContext`, rendu, flags rebuild, gestion modèle/GPU |
| `printable.rs` | Export PDF/SVG/PNG (pages imprimables) |
| `util_gl.rs` | Vertex types, uniforms, macros OpenGL |
| `util_3d.rs` | **Type aliases** : `Vector2/3`, `Matrix2/3/4`, `Quaternion`, `Point2/3` (tous `cgmath<f32>`) |
| `pdf_metrics.rs` | Métriques Helvetica pour texte dans les PDF |
| `config.rs` | Configuration utilisateur persistée en JSON |
| `paper/` | Modèle de données pur (Papercraft, Island, PaperOptions, I/O) |
| `paper/model/import/` | Importeurs : OBJ, GLTF, STL, Pepakura |

### Types centraux

**`GlobalContext`** (`main.rs`) — Conteneur maître de l'application
- Implémente `easy_imgui_window::Application`
- Champs clés : `gl: GlContext`, `data: PapercraftContext`, `rebuild: RebuildFlags`
- Gère : dialogs fichiers, messages d'erreur, config, version check

**`PapercraftContext`** (`ui.rs`) — Couche données + GPU
- Champs clés : `papercraft: Papercraft`, `gl_objs: GLObjects`, `ui: UiSettings`
- Gère : sélection, historique undo, positions des îles, grab states

**`RebuildFlags`** (`ui.rs`) — Dirty flags bitflags pour rendu incrémental
- `self.add_rebuild(RebuildFlags::SCENE_FBO | RebuildFlags::PAPER)` pour signaler un changement
- Checké dans la boucle de rendu pour ne reconstruire que ce qui est nécessaire

**`FileAction`** (`main.rs`) — Enum des opérations fichier
```rust
enum FileAction {
    OpenCraft, OpenCraftReadOnly, SaveAsCraft,
    ImportModel, UpdateObj, ExportObj,
    GeneratePrintable(FileFormat),
    Generate3dPdf,  // ← ajouté dans ce fork
}
```
Pour ajouter une action : implémenter `title()` et `is_save()`, ajouter le dispatch dans `do_file_action()`.

---

## Patterns importants

### Ajouter un item de menu
```rust
// Dans la fonction qui gère les menus (chercher les autres ui.menu_item_config)
if ui.menu_item_config(lbl(tr!("Mon item..."))).build() {
    menu_actions.mon_flag = true;
}
// Plus loin, dans le traitement des menu_actions :
if menu_actions.mon_flag {
    // ouvrir un file dialog ou agir directement
}
```

### Rendu OpenGL off-screen (FBO)
Pattern utilisé dans `create_thumbnail()` et `generate_3d_pdf_views()` :
```rust
let fbo  = glr::Framebuffer::generate(&self.gl)?;
let rbo  = glr::Renderbuffer::generate(&self.gl)?;
let rboz = glr::Renderbuffer::generate(&self.gl)?;

let fb_binder = BinderFramebuffer::bind(&fbo);
unsafe {
    let rb_binder = glr::BinderRenderbuffer::bind(&rbo);
    self.gl.renderbuffer_storage(rb_binder.target(), glow::RGBA8, W, H);
    self.gl.framebuffer_renderbuffer(fb_binder.target(), glow::COLOR_ATTACHMENT0, glow::RENDERBUFFER, Some(rbo.id()));
    rb_binder.rebind(&rboz);
    self.gl.renderbuffer_storage(rb_binder.target(), glow::DEPTH_COMPONENT, W, H);
    self.gl.framebuffer_renderbuffer(fb_binder.target(), glow::DEPTH_ATTACHMENT, glow::RENDERBUFFER, Some(rboz.id()));

    // Flip Y pour image droite
    self.data.ui.trans_scene.persp.y.y *= -1.0;
    self.gl.front_face(glow::CW);

    let thumb_data = self.data.prepare_thumbnail(Vector2::new(W as f32, H as f32));
    self.data.pre_render(RebuildFlags::all(), &TextBuilderDummy);
    self.gl.viewport(0, 0, W, H);
    self.gl.clear_color(1.0, 1.0, 1.0, 1.0);
    self.gl.clear_depth_f32(1.0);
    self.gl.clear(glow::COLOR_BUFFER_BIT | glow::DEPTH_BUFFER_BIT);
    self.render_scene(1.0);

    // Lire les pixels
    let mut pixbuf = image::RgbaImage::new(W as u32, H as u32);
    self.gl.read_pixels(0, 0, W, H, glow::RGBA, glow::UNSIGNED_BYTE,
        glow::PixelPackData::Slice(Some(&mut pixbuf)));

    self.gl.front_face(glow::CCW);
    self.data.restore_thumbnail(thumb_data);
}
self.add_rebuild(RebuildFlags::all());
```

### Changer la rotation de la caméra (vue 3D)
```rust
use cgmath::Quaternion;
// Les types locaux sont dans util_3d.rs : type Quaternion = cgmath::Quaternion<f32>
self.data.ui.trans_scene.rotation =
    Quaternion::from_axis_angle(Vector3::new(0.0, 1.0, 0.0), Deg(90.0_f32));
self.data.ui.trans_scene.recompute_obj();
```

### Générer un PDF avec lopdf
Voir `printable.rs:generate_pdf()` et `main.rs:generate_3d_pdf_views()` pour des exemples complets. Pattern de base :
```rust
use lopdf::{Document, Object, Stream, dictionary, xref::XrefType, content::{Content, Operation}};

let mut doc = Document::with_version("1.4");
doc.reference_table.cross_reference_type = XrefType::CrossReferenceTable;

// Compression parallèle des images via rayon (voir printable.rs)
// MediaBox : utiliser des types explicites pour éviter l'ambiguïté
"MediaBox" => vec![0i32.into(), 0i32.into(), page_w.into(), page_h.into()],
```

### Types — pièges fréquents
- `Vector3` dans `main.rs` = `cgmath::Vector3<S>` (générique). Pour f32, utiliser `type V3 = cgmath::Vector3<f32>` ou les alias de `util_3d`.
- `Vector2` dans `main.rs` = `easy_imgui::Vector2` (= `cgmath::Vector2<f32>`)
- `0.into()` dans un `vec![]` lopdf peut être ambigu → écrire `0i32.into()` ou `0.0_f32.into()`
- Itérer sur `&[(&str, ...)]` avec `.iter()` donne `&&str` → utiliser `for &(label, ref data) in slice.iter()`

---

## Localisation (i18n)

- Fichiers `.po` dans `locales/`
- Macro `tr!("texte")` pour les chaînes traduisibles
- Généré au build via `include_po` → `OUT_DIR/locale/translators.rs`
- Initialisation dans `main()` via `translators::set_locale(...)`

---

## Fonctionnalités ajoutées dans ce fork

### Export PDF 3D views (`feature/3d-views-pdf`)
**Fichier** : `src/main.rs` — fonction `generate_3d_pdf_views()`

Génère un PDF A4 paysage avec 6 vues du modèle assemblé (Front/Back/Left/Right/Top/Bottom) en grille 3×2 pour aider à l'assemblage.

- Menu : *File → Export 3D Views PDF...*
- Rendu : FBO OpenGL 512×512 px par vue, 6 rotations de caméra
- PDF : lopdf, compression parallèle rayon, titre centré
- Fond blanc, lignes de découpe en bleu (couleur hardcodée pour le PDF, restaurée après)
- Pas de labels de vue (Front/Back/…) dans le PDF final
- Scale auto-fit exact par vue : formule perspective `S = fill * camera_dist / (|rv.x| * focal + fill * rv.z)`, minimum sur tous les sommets → modèle remplit ~92% de chaque cellule sans déborder

#### Points d'attention pour modifier cette fonction

**Couleur des lignes de découpe**
`prepare_thumbnail()` désactive `show_3d_lines`. Il faut le réactiver juste après :
```rust
let thumb_data = self.data.prepare_thumbnail(...);
self.data.ui.show_3d_lines = true;
```
La couleur est injectée via `options_mut().line3d_cut.color` avant `pre_render`, puis restaurée après le rendu.

**Auto-scale perspective-correct**
Pour chaque vue, après avoir fixé la rotation :
```rust
// Vertices normalisés = après obj (centrage + normalisation à 1 unité), avant rotation/scale
// rv = rot_mat * v_normalized
// S = fill * camera_dist / (|rv.proj| * focal + fill * rv.z)   (min sur tous les sommets)
let tight_scale = normalized_verts.iter().fold(f32::INFINITY, |min_s, v| {
    let rv = rot_mat * *v;
    let lim = |proj: f32| {
        let d = proj.abs() * focal + FILL * rv.z;
        if d > 0.0 { FILL * camera_dist / d } else { f32::INFINITY }
    };
    min_s.min(lim(rv.x)).min(lim(rv.y))
});
self.data.ui.trans_scene.scale = tight_scale;
self.data.ui.trans_scene.recompute_obj();
```
`focal = persp[1][1]`, `camera_dist = -location.z = 30.0`.

**Nouvelles méthodes ajoutées**
- `Papercraft::options_mut()` (`paper/craft.rs`) — accès mutable aux options sans passer par `set_options`
- `PapercraftContext::papercraft_mut()` (`ui.rs`) — accès mutable au papercraft sous-jacent

---

## Build script (`build.rs`)

1. **Ressources Windows** : compile `res/resource.rc` si `WINDRES` ou `RC` est défini (skip gracieux sinon)
2. **Métriques Helvetica** : parse `thirdparty/afm/Helvetica.afm` → génère `helvetica_afm.rs`
3. **Locales** : génère `locale/translators.rs` depuis les fichiers `.po`
