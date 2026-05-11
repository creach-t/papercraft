# CLAUDE.md — Guide pour agents IA

Ce fichier aide les agents IA à comprendre rapidement l'architecture du projet et à contribuer efficacement.

---

## Présentation du projet

**Papercraft** est un outil graphique desktop (Windows/Linux) pour déplier des modèles 3D en patrons imprimables (PDF, SVG, PNG). Il est écrit en **Rust** avec une interface ImGui et un rendu OpenGL.

- Fork de travail : https://github.com/creach-t/papercraft
- Original upstream : https://github.com/rodrigorc/papercraft
- Branche principale du fork : `main` (toutes les features y sont mergées)
- Branches feature archivées : `feature/3d-views-pdf`, `feature/interactive-3d-pdf`, `feature/auto-unfold`, `feature/label-etiquette`
- Version : 2.11.0

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
| `main.rs` (≈4929 l.) | Point d'entrée, boucle événements, `GlobalContext`, actions fichier, exports |
| `ui.rs` (≈1900 l.) | `PapercraftContext`, rendu, flags rebuild, gestion modèle/GPU |
| `printable.rs` (≈650 l.) | Export PDF/SVG/PNG (pages imprimables) |
| `util_gl.rs` | Vertex types (`MVertex3D`, `MVertex2D`, etc.), uniforms, macros OpenGL |
| `util_3d.rs` | **Type aliases** : `Vector2/3`, `Matrix2/3/4`, `Quaternion`, `Point2/3` (tous `cgmath<f32>`) |
| `pdf_metrics.rs` | Métriques Helvetica pour texte dans les PDF |
| `config.rs` | Configuration utilisateur persistée en JSON |
| `paper.rs` | Ré-exports publics du module `paper::` |
| `paper/craft.rs` (≈850 l.) | Struct `Papercraft`, options, îles, edges, labels utilisateur, noms d'îles |
| `paper/craft/file.rs` | I/O sérialisation `.craft` (ZIP + JSON) |
| `paper/craft/update.rs` | Mise à jour du modèle depuis imports OBJ/GLTF |
| `paper/model.rs` (≈600 l.) | Struct `Model` (vertices, edges, faces, textures), calculs géométriques |
| `paper/model/formats.rs` | Dispatcher import/export |
| `paper/model/formats/{gltf,stl,waveobj,pepakura}/` | Importeurs spécialisés |
| `viewer3d_template.html` (442 l.) | Template WebGL autonome (0 dépendance externe) |

### Types centraux

**`GlobalContext`** (`main.rs`) — Conteneur maître de l'application
- Implémente `easy_imgui_window::Application`
- Champs clés : `gl: GlContext`, `data: PapercraftContext`, `rebuild: RebuildFlags`
- Gère : dialogs fichiers, messages d'erreur, config, version check

**`PapercraftContext`** (`ui.rs:159`) — Couche données + GPU
```rust
pub struct PapercraftContext {
    papercraft: Papercraft,
    gl_objs: GLObjects,
    undo_stack: Vec<Vec<UndoAction>>,
    selected_face: Option<FaceIndex>,
    selected_edges: Option<FxHashSet<EdgeIndex>>,
    selected_islands: Vec<IslandFaceKey>,
    grabbed_label: Option<(LabelKey, Vector2)>,  // label en cours de drag
    pub ui: UiSettings,
}
```

**`UiSettings`** (`ui.rs:192`)
```rust
pub struct UiSettings {
    pub mode: MouseMode,
    pub trans_scene: Transformation3D,
    pub trans_paper: TransformationPaper,
    pub show_textures: bool,
    pub show_flaps: bool,
    pub show_3d_lines: bool,   // ← à réactiver après prepare_thumbnail()
    pub xray_selection: bool,
    pub show_texts: bool,
    pub highlight_overlaps: bool,
    pub draw_paper: bool,
}
```

**`Transformation3D`** (`ui.rs:208`) — Caméra vue 3D
```rust
pub struct Transformation3D {
    pub location: Vector3,
    pub rotation: Quaternion,
    pub scale: f32,
    pub obj: Matrix4,    // normalisation modèle (centrage + scale à 1 unité)
    pub persp: Matrix4,  // focal = persp[1][1]
    pub view: Matrix4,
    pub mnormal: Matrix3,
}
```

**`RebuildFlags`** (`ui.rs`) — Dirty flags bitflags pour rendu incrémental
- `self.add_rebuild(RebuildFlags::SCENE_FBO | RebuildFlags::PAPER)` pour signaler un changement
- Checké dans la boucle de rendu pour ne reconstruire que ce qui est nécessaire

**`FileAction`** (`main.rs:441`) — Enum des opérations fichier
```rust
enum FileAction {
    OpenCraft,
    OpenCraftReadOnly,
    SaveAsCraft,
    ImportModel,
    UpdateObj,
    ExportObj,
    GeneratePrintable,
    Generate3dPdf,              // ← ajouté dans ce fork
    GenerateInteractive3dPdf,   // ← ajouté dans ce fork
}
```
Pour ajouter une action : implémenter `title()` et `is_save()`, ajouter le dispatch dans `do_file_action()`.

**`Label`** (`paper/craft.rs:119`) — Labels utilisateur libres (pas les vues 3D)
```rust
pub struct Label {
    pub pos: Vector2,    // coordonnées papier en mm
    pub size: Vector2,
    pub title: String,
}
```
Sérialisés dans `.craft`, éditables dans l'UI. Itérateur : `papercraft.labels()`.

**Système de label thumbnail** (`main.rs`, `ui.rs`, `printable.rs`)
- Créé via *Edit → Add label* : place un label en haut de la page 1, titre = nom du fichier modèle
- La texture thumbnail (`GLObjects.label_thumbnail_tex`) est une render OpenGL 256×256 du modèle (vue isométrique), créée par `create_label_thumbnail()` (`main.rs:3392`)
- Layout du label : thumbnail carrée à gauche | titre + dimensions à droite
  - Titre centré à 38% de la hauteur, taille police 0.28×hauteur
  - Dimensions `W:/H:/D: mm` (bounding box × scale) à 65% de la hauteur, taille 0.16×hauteur
- Rendu dans `ui.rs:1268` (OpenGL) et `printable.rs:1145` (PDF/SVG vecteur)

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

- Fichiers `.po` dans `locales/` (EN, ES)
- Macro `tr!("texte")` pour les chaînes traduisibles
- Généré au build via `include_po` → `OUT_DIR/locale/translators.rs`
- Initialisation dans `main()` via `translators::set_locale(...)`

---

## Fonctionnalités ajoutées dans ce fork

### Export PDF 6 vues (`feature/3d-views-pdf`)
**Fichier** : `src/main.rs:3511-3825` — fonction `generate_3d_pdf_views()`

Génère un PDF A4 paysage (841.89×595.28 pts) avec 6 vues du modèle assemblé (grille 3×2).

- Menu : *File → Export 3D Views PDF...*
- Rendu : FBO OpenGL 512×512 px par vue, 6 rotations de caméra
- PDF : lopdf, compression parallèle rayon, titre centré (Helvetica 14pt)
- Fond blanc, arêtes de découpe en bleu hardcodé `(0.18, 0.45, 1.0, 1.0)`

#### Vues et rotations (`main.rs:3523-3532`)
```rust
let views: &[(&str, Option<(V3, f32)>)] = &[
    ("Front",  None),
    ("Back",   Some((V3::new(0.0, 1.0, 0.0), 180.0))),
    ("Right",  Some((V3::new(0.0, 1.0, 0.0), -90.0))),
    ("Left",   Some((V3::new(0.0, 1.0, 0.0),  90.0))),
    ("Top",    Some((V3::new(1.0, 0.0, 0.0), -90.0))),
    ("Bottom", Some((V3::new(1.0, 0.0, 0.0),  90.0))),
];
```
Les strings ("Front", "Back"…) sont définies ici. Si les labels sont affichés dans le PDF, c'est via ces strings — vérifier dans la fonction si elles sont passées à lopdf.

#### Disposition PDF
- Margin : 18 pt, Gap : 8 pt entre cellules
- Titre en haut (22 pt de hauteur)
- Cellules carrées, grille 3 cols × 2 rangées

#### Points d'attention

**Couleur des arêtes de découpe** — `prepare_thumbnail()` désactive `show_3d_lines`. Réactiver juste après :
```rust
let thumb_data = self.data.prepare_thumbnail(...);
self.data.ui.show_3d_lines = true;
```
Override couleur avant `pre_render`, restaurer après :
```rust
self.data.papercraft_mut().options_mut().line3d_cut.color =
    imgui::Color::new(0.18, 0.45, 1.0, 1.0);
// ... rendu ...
self.data.papercraft_mut().options_mut().line3d_cut.color = orig_cut_color;
```

**Auto-scale perspective-correct** (`main.rs:3607-3630`)
```rust
// focal = persp[1][1], camera_dist = 30.0, fill = 0.92
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
Modèle remplit ~92% de chaque cellule sans déborder.

---

### Export visionneuse 3D interactive (.html) (`feature/interactive-3d-pdf`)
**Fichiers** : `src/main.rs:3829-3985` — `generate_interactive_3d_pdf()` + `src/viewer3d_template.html`

Génère un fichier `.html` autonome (zéro dépendance externe) avec une visionneuse 3D WebGL.

- Menu : *File → Export Interactive 3D Viewer (.html)...*
- Contrôles : left-drag rotation, right-drag pan, scroll zoom, touch support
- Arêtes de découpe affichées en bleu (#2488FF)
- Survol pièce → highlight jaune dans vue 3D + panneau 2D (layout plat + nom de la pièce)

#### Noms d'îles
```rust
self.data.papercraft_mut().rebuild_island_names();  // assigne A, B, C, ..., Z, AA, AB, ...
```
`rebuild_island_names()` (`paper/craft.rs:752`) : trie les îles par aire décroissante.

#### Architecture des données exportées

| Variable JS | Contenu | Stride |
|---|---|---|
| `VERTS` | `Float32Array` — `[x3d, y3d, z3d, r, g, b, island_f]` × 3 vertices/face | 7 floats = 28 bytes |
| `EDGES` | `Float32Array` — `[x3d, y3d, z3d]` × 2 endpoints/arête coupée | 3 floats = 12 bytes |
| `ISLANDS` | JS array — `{name: str, flat: [...]}` par île | — |
| `ISLANDS[i].flat` | `[x0,y0, x1,y1, x2,y2, r,g,b, ...]` — 9 floats/triangle | 9 floats |

Couleur des faces : échantillonnée à l'UV centre de chaque face, sinon gris (0.78, 0.78, 0.78).

#### Remplacement dans le template
```rust
let html = include_str!("viewer3d_template.html")
    .replace("__TITLE__", &title)
    .replace("__VERTS__", &js_verts)
    .replace("__EDGES__", &js_edges)
    .replace("__ISLANDS__", &js_islands);
```
Pas de `format!` Rust — les accolades JS ne posent pas de problème.

#### Architecture `viewer3d_template.html`

**Shaders WebGL** :
- VS/FS mesh (`VS_MESH`/`FS_MESH_D`) : lighting via dérivées (`GL_OES_standard_derivatives`), fallback flat shading
- VS/FS edge (`VS_EDGE`/`FS_EDGE`) : lignes bleu `#2488FF`
- Highlight survol : `mix(aCol, aCol + vec3(0.28, 0.28, 0.06), h)` (teinte jaune si island hover)

**Raycast CPU — Möller–Trumbore** (ligne ≈194) :
- Parcourt tous les triangles de `VERTS` à chaque `mousemove`
- Ray vers model space : `rot^T × ray_eye`, origine : `rot^T × (-panX, -panY, zoom)`

**Panneau 2D** (ligne ≈255) :
- Canvas 2D dédié, fit per-island dans 220×220px
- Dessine triangles colorées + arêtes noires/blanches selon luminance

**High-DPI** :
```javascript
const dpr = window.devicePixelRatio || 1;
canvas.width = Math.round(canvas.clientWidth * dpr);
```

---

## Nouvelles méthodes ajoutées dans ce fork

| Méthode | Fichier | Rôle |
|---|---|---|
| `Papercraft::options_mut()` | `paper/craft.rs:568` | Accès mutable aux options sans passer par `set_options` |
| `Papercraft::rebuild_island_names()` | `paper/craft.rs:752` | Assigne noms A/B/C… aux îles par aire décroissante |
| `PapercraftContext::papercraft_mut()` | `ui.rs:594` | Accès mutable au `Papercraft` sous-jacent |

---

## Build script (`build.rs`)

1. **Ressources Windows** : compile `res/resource.rc` si `WINDRES` ou `RC` est défini (skip gracieux sinon)
2. **Métriques Helvetica** : parse `thirdparty/afm/Helvetica.afm` → génère `helvetica_afm.rs`
3. **Locales** : génère `locale/translators.rs` depuis les fichiers `.po`

---

## Dépendances clés (`Cargo.toml`)

| Crate | Version | Rôle |
|---|---|---|
| `cgmath` | 0.18 | Vecteurs/matrices (`Vector3`, `Matrix4`, `Quaternion`) |
| `easy-imgui-window` | 0.22.0 | Fenêtre + ImGui + OpenGL context |
| `easy-imgui-filechooser` | 0.5 | File dialogs |
| `lopdf` | 0.40 | Génération PDF |
| `image` | 0.25 | PNG/JPEG + `RgbaImage` |
| `rayon` | 1 | Compression images en parallèle |
| `zip` | 8.1.0 | Format `.craft` (ZIP + JSON) |
| `serde` | 1 | Sérialisation modèles |
| `slotmap` | 1 | `IslandKey`, `LabelKey`, `EdgeIndex`, etc. |
| `tr` | 0.1.10 | Macro `tr!(...)` pour i18n |

---

## TODOs connus

| Fichier | Note |
|---|---|
| `main.rs:902` | `//TODO: list third party SW` (boîte About) |
| `paper/craft.rs:440` | `#[serde(default)] //TODO: default not actually needed` |
| `ui.rs` | `//TODO PrintableTexts duplicated here and in generate_pages???` |
| `paper/model/formats/pepakura/data.rs:167` | `//TODO mbcs?` (multi-byte charset, héritage) |
| `printable.rs:63` | `_edge_id_position` préfixé underscore — variable lue mais pas encore utilisée dans cette fonction |
