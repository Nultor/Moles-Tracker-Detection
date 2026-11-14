# Mole Tracker — Build Plan for Claude Code

> Personal tool to photograph/film my body, detect each mole, measure its **position, size, colour and shape**, store everything, and show how each mole **evolves over time**.
> This is a *tracking* tool, not a diagnostic tool. No melanoma-risk prediction in v1.

---

## 0. Instructions for Claude Code

- Build in the milestone order below. Finish and test each milestone before starting the next.
- Keep every model **pluggable** behind a small interface (detector, segmenter) so I can swap models later.
- All personal data stays **local**: `data/` is git-ignored, no images in the repo, no cloud uploads.
- Ask me before downloading large model weights (> 500 MB) or adding heavy dependencies.
- Write unit tests with **synthetic images** (see §9) so measurements are verified against known ground truth.
- Python 3.11, managed with `uv` (or `pip` + `pyproject.toml`).
- Database access only through **SQLAlchemy 2.0 ORM** (`moletracker/db/`) on **PostgreSQL**, schema changes only through **Alembic** migrations.

### Decisions log

| Date | Decision | Why | Alternatives rejected |
|---|---|---|---|
| 2026-11-14 | **Local only** for now: Postgres, API and frontend all run on my machine; no cloud deployment. | Body photos are sensitive health data; keep the privacy promise in §10. | GCP deployment (revisit later with security-auth-expert + deployment-expert). |
| 2026-11-14 | **Next.js** frontend (`web/`) + **FastAPI** backend API (`moletracker/api/`). | Next.js gives more UI flexibility than Streamlit; a Next.js frontend needs a Python API to call the pipeline, and FastAPI is the standard typed choice. | Streamlit (originally planned). |
| 2026-11-14 | **PostgreSQL** via SQLAlchemy 2.0 + Alembic. | Real server DB: proper enums, `JSONB`, `TIMESTAMPTZ`, concurrent access from the API; same DB engine if deployed later. | SQLite (originally planned). |

---

## 1. Tech stack

| Purpose | Library |
|---|---|
| Video/photo I/O, ArUco markers, contours | `opencv-contrib-python` |
| Arrays, maths | `numpy`, `scipy` |
| Colour spaces (Lab, ΔE2000), shape metrics | `scikit-image` |
| Database | **PostgreSQL** (local, via Docker Compose) + **SQLAlchemy 2.0 ORM** (typed `Mapped[...]` models) + **Alembic** migrations, driver `psycopg` (v3) |
| Backend API | `fastapi` + `uvicorn`, `pydantic` schemas |
| Tables / charts | `pandas` (backend); charts in the frontend (e.g. `plotly.js` / Recharts) |
| UI (upload, review, confirm, timelines) | **Next.js** (App Router, TypeScript) in `web/` |
| Detection (optional extra) | existing repo below, or `ultralytics` YOLO |
| Segmentation (optional extra) | `segment-anything` / MedSAM / skin-specific SAM, `torch` |

---

## 2. Project structure

```
mole-tracker/
├── pyproject.toml
├── README.md
├── compose.yaml               # local PostgreSQL only (bound to 127.0.0.1)
├── .env.example               # DATABASE_URL etc. (real .env is git-ignored)
├── config.yaml                # marker size, thresholds, body regions, model choices
├── assets/
│   ├── aruco_marker_20mm.pdf  # generated, printable
│   └── body_regions.yaml      # fixed list of regions
├── moletracker/
│   ├── capture.py             # video → best frames, photo loading, EXIF
│   ├── calibration.py         # ArUco detection, mm/px, perspective, colour correction
│   ├── detect/
│   │   ├── base.py            # Detector interface -> list[BBox]
│   │   ├── blob.py            # classical OpenCV baseline (no ML)
│   │   └── repo_model.py      # wrapper for the external repo model
│   ├── segment/
│   │   ├── base.py            # Segmenter interface: (image, bbox) -> mask
│   │   ├── otsu.py            # classical baseline
│   │   └── sam.py             # SAM-family with box prompt
│   ├── measure.py             # size, shape, colour features
│   ├── register.py            # align a new session to reference map
│   ├── match.py               # Hungarian matching old ↔ new moles
│   ├── contours.py            # contour extraction, alignment, overlay metrics + rendering
│   ├── compare.py             # change detection, alerts
│   ├── db/
│   │   ├── models.py          # SQLAlchemy ORM models (§5)
│   │   ├── session.py         # engine + session factory (PostgreSQL)
│   │   └── repository.py      # query/helper functions used by pipeline + API
│   ├── api/
│   │   ├── main.py            # FastAPI app (binds to 127.0.0.1)
│   │   ├── schemas.py         # pydantic request/response models
│   │   └── routes/            # sessions, images, moles, matches, timelines, files
│   ├── alembic/               # migrations (alembic.ini at project root)
│   └── pipeline.py            # end-to-end: images → observations
├── web/                       # Next.js frontend (App Router, TypeScript)
├── scripts/
│   ├── make_marker.py         # print the ArUco + colour patch sheet
│   └── ingest.py              # CLI: ingest a folder/video for a region
├── tests/
└── data/                      # git-ignored: db + images + crops
```

---

## 3. Capture protocol (the app must enforce/encourage this)

Consistency of capture matters more than model quality.

1. **One body region per photo set** from a fixed list (`body_regions.yaml`), e.g. `upper_back`, `lower_back`, `chest`, `abdomen`, `left_upper_arm`, `left_forearm`, `right_thigh_front`… (~20 regions).
2. A **printed calibration sticker** in every image, on the skin near the moles:
   - one **ArUco marker** (DICT_4X4_50) with a known side length (default 20 mm),
   - next to it a small **colour reference strip**: white, mid-grey, black + optionally skin-tone patches (or a cheap commercial 24-patch colour card).
3. Same room, same light, phone held roughly perpendicular to the skin, flash consistently on or off, no beauty filters/HDR. If possible, lock exposure and white balance.
4. **Video is allowed**: the app extracts the sharpest frames (see §4.1). Photos are preferred for v1.

`scripts/make_marker.py` generates a printable PDF with several markers + colour strips at exact physical size.

---

## 4. Pipeline

### 4.1 Capture → frames (`capture.py`)
- Video: read with `cv2.VideoCapture`, sample ~3 fps, score sharpness with **variance of Laplacian**, discard frames where the ArUco marker is not detected, keep the top-N sharpest frames that are not near-duplicates.
- v1: the user picks / the app auto-picks **one best frame per region**. Merging detections across many frames (frame-to-frame tracking or stitching) is a later milestone (M7).

### 4.2 Calibration (`calibration.py`)
**Size / perspective**
- Detect the ArUco marker (`cv2.aruco.ArucoDetector`).
- Compute homography from the marker's 4 corners to a metric plane → **mm per pixel**, and optionally warp the image so the skin plane around the marker is fronto-parallel.
- Store `mm_per_px`, marker corners, and a **quality score** (marker size in px, viewing angle from homography). Reject or flag images where the marker is too small (< 80 px side) or the angle > ~30°.
- Known limit: the body is curved; measurements are only reliable near the marker. Store each mole's **distance to marker**; flag size measurements far from it as low-confidence.

**Colour**
- Sample the reference patches (positions known relative to the marker).
- Fit a **3×3 colour-correction matrix** (least squares, linearised sRGB) from measured → known patch values; apply to the image. Fallback: white-balance from the grey patch only.
- Convert to **CIELAB** (`skimage.color.rgb2lab`) for all colour measurements.

### 4.3 Detection (`detect/`)
Interface: `detect(image) -> list[BBox(x, y, w, h, score)]`

1. **Baseline (build first, no ML):** in Lab, find spots darker than local skin — difference-of-Gaussians / `cv2.SimpleBlobDetector` on the L channel, filter by size in mm (e.g. 1–15 mm), circularity, and contrast vs surrounding skin.
2. **External repo:** `https://github.com/RoujiaW/moles-detection` — **first inspect it**: what model (likely YOLO-based detection + a classifier, per the author's NVIDIA blog post), framework/version, available weights, licence, expected input (dermoscopy vs wide photo). Write a wrapper only if weights are available and it runs on wide-field body photos. Report findings to me before integrating.
3. **Reference paper — DiaMole** (Liu et al., J Healthc Eng 2021, doi:10.1155/2021/6698176): Python software that does detection (best: Faster R-CNN, mAP 0.835) → segmentation on mobile-phone mole images, with manual annotation of moles in the software. Use it as a **design reference and baseline** to compare against, not as a dependency: it was built for close-up photos of single moles (MoleMapper app images, downscaled to 320×568 px) on Python 3.6. In M0, check whether its code/weights are publicly available and report back. Also look at the MoleMapper dataset access conditions (Synapse) as a possible extra source of phone images for training/evaluation.
4. **Later:** fine-tune `ultralytics` YOLO on the iToBoS dataset (ISIC Archive, body-photo patches with mole bounding boxes) + my own labelled photos.

### 4.4 Segmentation (`segment/`)
Interface: `segment(image, bbox) -> binary mask`
- **Baseline:** Otsu threshold on L* within the padded bbox crop, keep the largest connected component, fill holes.
- **SAM variant:** box-prompted SAM (generic SAM, MedSAM, or a skin-fine-tuned SAM; the exact checkpoint is configurable in `config.yaml`). The pattern "detector box → SAM mask" is well established for skin lesions.
- **LKM-SAM candidate** (Dai et al., *Pattern Analysis and Applications* 29, 205 (2026), doi:10.1007/s10044-026-01733-9): SAM fine-tuned with LoRA, KAN convolutions in the encoder neck, and a Mask2Former head instead of the SAM decoder, so it segments **without prompts**. It was trained and evaluated on **dermoscopic close-ups** (ISIC 2017/2018, PH²). Integration rules:
  - In M0, check whether code and weights are public (paper page, GitHub, author contact). If they aren't, **do not reimplement it** in v1; use box-prompted SAM/MedSAM instead and revisit later.
  - If available, feed it the **padded crop around each detected box** (looks closest to a dermoscopic close-up), never the whole body photo.
  - Because it's prompt-free, if a crop contains several masks keep the one overlapping the detector box most.
  - Benchmark it against Otsu and box-prompted SAM on my own phone crops (I'll hand-outline ~30 moles as ground truth; report Dice/IoU) before making it the default.

### 4.5 Measurements (`measure.py`)
For each mole mask, compute and store:

**Position**
- `cx_px, cy_px` in the image
- `x_mm, y_mm` relative to the ArUco marker (via homography)
- `region` (from capture)
- position in the region's **reference frame** after registration (§4.6)

**Size** (using `mm_per_px`)
- `area_mm2`
- `max_diameter_mm` (max Feret), `min_diameter_mm`, `equiv_diameter_mm`
- `size_confidence` (from marker distance + angle)

**Shape**
- `asymmetry`: 1 − IoU of the mask with itself flipped across its principal axes (take the worse of both axes)
- `border_irregularity`: perimeter² / (4π·area) (1.0 = perfect circle)
- `eccentricity`, `solidity`

**Colour** (on colour-corrected Lab image)
- `L_mean, a_mean, b_mean`, and their std devs
- `skin_L, skin_a, skin_b` from a ring 1–3 mm around the mask
- `deltaE_vs_skin` (CIEDE2000): more robust to lighting than absolute colour
- `n_colour_clusters`: k-means in Lab on mole pixels; count clusters separated by ΔE > threshold
- `dominant_colours`: JSON list of Lab cluster centres + proportions

Save a fixed-size crop (e.g. 256×256, centred, scaled to a fixed mm/px) so crops are visually comparable over time. The outline itself is stored and compared as described in §4.9.

### 4.6 Registration — aligning sessions (`register.py`)
Goal: put moles from today's photo into the same coordinate frame as the region's **reference map**.
1. The first good session of a region becomes its **reference map** (mole positions in mm).
2. For a new session of that region: we have mole centroids in mm (marker frame). The marker won't be placed at exactly the same spot, so estimate a **similarity transform** (rotation + translation; scale ≈ 1 because both are in mm) between the new point set and the reference point set with **RANSAC** (try pairs of points, count inliers within ~3 mm).
3. Optional assist: the user confirms 2–3 "anchor moles" in the UI to seed the transform.

### 4.7 Matching (`match.py`)
- Cost matrix between reference moles and new detections: `cost = position_distance_mm + w1·|Δ log area| + w2·ΔE_mean`.
- Solve with `scipy.optimize.linear_sum_assignment`; reject pairs with distance > gate (e.g. 5 mm).
- Unmatched new → **"possible new mole"**; unmatched reference → **"not seen"** (could be out of frame).
- **Human-in-the-loop:** the UI shows proposed matches; I confirm/correct. Confirmed matches are stored as ground truth (and later used to tune the weights).

### 4.8 Change detection (`compare.py`)
For each mole, compare the latest observation to the baseline and the previous one:
- area change %, max-diameter change (mm), ΔE2000 of mean colour, change in `n_colour_clusters`, change in asymmetry/border.
- **Noise floor first:** the app must support a "repeatability session" (3 photos of the same region on the same day). Use the observed spread to set thresholds so alerts only fire **above measurement noise** (e.g. change > max(configured threshold, 3σ)).
- Default starting thresholds in `config.yaml` (to be tuned): area +20 %, diameter +1 mm, ΔE > 8, any new colour cluster, any new mole.
- Alert text is neutral: "changed beyond measurement noise — consider showing a dermatologist".

### 4.9 Contours & overlay comparison (`contours.py`)
Size numbers alone hide *where* a mole changed. Every observation also stores the mole's **outline**, so the same mole from different dates can be laid on top of each other.

**Storing the contour**
- From the mask, extract the outer contour (`cv2.findContours`, `CHAIN_APPROX_NONE`), convert pixels → **mm** with the image homography, express it **relative to the mole's centroid**, and rotate it into the **region reference frame** (using the registration transform from §4.6) so all dates share the same orientation.
- Resample to a fixed **256 points**, evenly spaced along the perimeter, starting at angle 0° and going counter-clockwise. A fixed format makes comparisons and storage simple.
- Also save a **normalised crop + mask**: re-rendered at a fixed scale (e.g. 0.05 mm/px), centred on the centroid and rotated to the reference frame. Crops from any date are then pixel-aligned and can be overlaid directly.

**Aligning two dates of the same mole**
1. Start from the region registration (§4.6): centroids on top of each other, same orientation.
2. Refine with a **rigid** alignment only (small rotation ± 15°, translation ± 1 mm) that maximises mask IoU, or ICP on the contour points.
3. **Never scale** during alignment: scale comes from the marker, and a change in scale *is* the growth we want to measure.

**Change metrics** (between any two observations, normally baseline → latest and previous → latest)
- **IoU / Dice** of the aligned masks (1.0 = identical shape and size).
- **Grown area** (mm², new pixels outside the old outline) and **shrunk area** (old pixels no longer covered).
- **Hausdorff distance** and **mean contour distance** in mm (how far the outline moved, worst case and average).
- **Radial profile**: distance from centroid to the outline at 360 angles, for each date. The difference shows growth **per direction**: max radial growth (mm) and the angle where it happens. Growth concentrated on one side is flagged even if total area barely changed.
- **Shape-only change**: compare Fourier descriptors of size-normalised contours, to separate "same shape, bigger" from "shape changing".

**Visualisation**
- Overlay on the latest normalised crop: old outline (one colour), new outline (another), grown area shaded, shrunk area hatched.
- Time slider / small-multiples strip: every dated outline of the mole, aligned, with a 1 mm scale bar.
- Polar plot of the radial profiles per date.

**Noise floor**
Repeatability sessions (§4.8) also give the normal same-day spread of IoU, Hausdorff and radial difference, so contour alerts only fire above measurement noise. The outline is the most noise-sensitive measurement (lighting, blur, segmentation), so contour alerts use the **noise floor + a margin** and require the change to show up in **two consecutive sessions** before alerting, unless it is large.

---

## 5. Database — SQLAlchemy 2.0 ORM (PostgreSQL)

### 5.1 Rules
- Use **SQLAlchemy 2.0 style**: `DeclarativeBase`, `Mapped[...]`, `mapped_column`, `select()`; no legacy `Query` API.
- PostgreSQL runs **locally** in Docker Compose (`compose.yaml`, port bound to `127.0.0.1` only, data in a named volume). Connection string from the `DATABASE_URL` env var (`postgresql+psycopg://moles:…@127.0.0.1:5432/moles`), read from a git-ignored `.env`; `.env.example` is committed without real secrets.
- `session.py`: one engine (`pool_pre_ping=True`), `sessionmaker(expire_on_commit=False)`.
- Provide a `get_session()` context manager (and a FastAPI dependency wrapping it); the pipeline and API never create engines themselves.
- All queries live in `repository.py` (e.g. `get_mole_timeline(mole_id)`, `get_reference_map(region)`, `add_observation(...)`, `confirm_match(obs_id, mole_id)`), so API routes stay thin.
- **Alembic** from day one: initial migration generated from the models; every model change = new migration (`alembic revision --autogenerate`), reviewed by hand (autogenerate misses some things, e.g. enum value changes). Never edit the DB by hand.
- Store file paths **relative to `data/`** so the data folder can be moved. Images stay on disk in `data/`, not in the DB.
- JSON fields (bbox, homography, colour matrix, dominant colours, quality flags, contours) use PostgreSQL **`JSONB`**.
- Timestamps as `DateTime(timezone=True)` (`TIMESTAMPTZ`, UTC) in the DB, shown in local time in the UI.
- Enums are native PostgreSQL enum types (`MoleStatus`, `MatchStatus`).
- Tests run against a separate local **`moles_test`** database: schema created once per test run (`alembic upgrade head`), each test wrapped in a transaction that is rolled back, so tests stay isolated and fast. Tests never touch the real `moles` database.

### 5.2 Models (`db/models.py`)

```python
from __future__ import annotations
from datetime import datetime, timezone
from enum import Enum
from typing import Any, Optional
from sqlalchemy import DateTime, ForeignKey, String, Text, UniqueConstraint
from sqlalchemy import Enum as SAEnum
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


def utcnow() -> datetime:
    return datetime.now(timezone.utc)


class Base(DeclarativeBase):
    type_annotation_map = {
        dict[str, Any]: JSONB,
        list[Any]: JSONB,
        datetime: DateTime(timezone=True),   # TIMESTAMPTZ
    }


class MoleStatus(str, Enum):
    active = "active"
    watch = "watch"
    removed = "removed"


class MatchStatus(str, Enum):
    unmatched = "unmatched"
    auto = "auto"
    confirmed = "confirmed"
    new = "new"
    rejected = "rejected"


class CaptureSession(Base):            # "Session" would clash with SQLAlchemy's Session
    __tablename__ = "capture_sessions"
    id: Mapped[int] = mapped_column(primary_key=True)
    taken_at: Mapped[datetime] = mapped_column(default=utcnow)
    is_repeatability: Mapped[bool] = mapped_column(default=False)  # noise-floor sessions
    notes: Mapped[Optional[str]] = mapped_column(Text)
    images: Mapped[list[Image]] = relationship(back_populates="session", cascade="all, delete-orphan")


class Image(Base):
    __tablename__ = "images"
    id: Mapped[int] = mapped_column(primary_key=True)
    session_id: Mapped[int] = mapped_column(ForeignKey("capture_sessions.id"), index=True)
    region: Mapped[str] = mapped_column(String(50), index=True)
    path: Mapped[str] = mapped_column(String(500))           # relative to data/
    source: Mapped[str] = mapped_column(String(20))          # photo | video_frame
    sharpness: Mapped[Optional[float]]
    mm_per_px: Mapped[Optional[float]]
    marker_corners: Mapped[Optional[list[Any]]]
    homography: Mapped[Optional[list[Any]]]                  # 3x3
    colour_matrix: Mapped[Optional[list[Any]]]               # 3x3
    quality_flags: Mapped[Optional[dict[str, Any]]]
    session: Mapped[CaptureSession] = relationship(back_populates="images")
    observations: Mapped[list[Observation]] = relationship(back_populates="image", cascade="all, delete-orphan")


class Mole(Base):
    __tablename__ = "moles"
    __table_args__ = (UniqueConstraint("label"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    region: Mapped[str] = mapped_column(String(50), index=True)
    label: Mapped[str] = mapped_column(String(30))           # e.g. "upper_back-07"
    ref_x_mm: Mapped[float]
    ref_y_mm: Mapped[float]
    first_seen_session_id: Mapped[Optional[int]] = mapped_column(ForeignKey("capture_sessions.id"))
    status: Mapped[MoleStatus] = mapped_column(SAEnum(MoleStatus), default=MoleStatus.active)
    notes: Mapped[Optional[str]] = mapped_column(Text)
    observations: Mapped[list[Observation]] = relationship(back_populates="mole")
    alerts: Mapped[list[Alert]] = relationship(back_populates="mole")


class Observation(Base):               # one mole seen in one image
    __tablename__ = "observations"
    id: Mapped[int] = mapped_column(primary_key=True)
    image_id: Mapped[int] = mapped_column(ForeignKey("images.id"), index=True)
    mole_id: Mapped[Optional[int]] = mapped_column(ForeignKey("moles.id"), index=True)  # NULL until matched
    # position
    bbox: Mapped[list[Any]]                                  # [x, y, w, h] px
    cx_px: Mapped[float]
    cy_px: Mapped[float]
    x_mm: Mapped[Optional[float]]                            # marker frame
    y_mm: Mapped[Optional[float]]
    reg_x_mm: Mapped[Optional[float]]                        # region reference frame
    reg_y_mm: Mapped[Optional[float]]
    dist_to_marker_mm: Mapped[Optional[float]]
    # size
    area_mm2: Mapped[Optional[float]]
    max_diameter_mm: Mapped[Optional[float]]
    min_diameter_mm: Mapped[Optional[float]]
    equiv_diameter_mm: Mapped[Optional[float]]
    size_confidence: Mapped[Optional[float]]
    # shape
    asymmetry: Mapped[Optional[float]]
    border_irregularity: Mapped[Optional[float]]
    eccentricity: Mapped[Optional[float]]
    solidity: Mapped[Optional[float]]
    # colour (CIELAB, after colour correction)
    L_mean: Mapped[Optional[float]]
    a_mean: Mapped[Optional[float]]
    b_mean: Mapped[Optional[float]]
    L_std: Mapped[Optional[float]]
    a_std: Mapped[Optional[float]]
    b_std: Mapped[Optional[float]]
    skin_L: Mapped[Optional[float]]
    skin_a: Mapped[Optional[float]]
    skin_b: Mapped[Optional[float]]
    deltaE_vs_skin: Mapped[Optional[float]]
    n_colour_clusters: Mapped[Optional[int]]
    dominant_colours: Mapped[Optional[list[Any]]]            # [{"lab": [L,a,b], "share": 0.4}, ...]
    # files + provenance
    contour_mm: Mapped[Optional[list[Any]]]                  # 256 [x, y] points, mm, centroid-relative, region frame
    contour_px: Mapped[Optional[list[Any]]]                  # raw contour in image pixels (for drawing on the photo)
    norm_scale_mm_per_px: Mapped[Optional[float]]            # scale of the normalised crop/mask
    crop_path: Mapped[Optional[str]] = mapped_column(String(500))       # raw crop
    mask_path: Mapped[Optional[str]] = mapped_column(String(500))       # raw mask
    norm_crop_path: Mapped[Optional[str]] = mapped_column(String(500))  # normalised, aligned crop
    norm_mask_path: Mapped[Optional[str]] = mapped_column(String(500))  # normalised, aligned mask
    detector: Mapped[str] = mapped_column(String(30))
    segmenter: Mapped[str] = mapped_column(String(30))
    match_status: Mapped[MatchStatus] = mapped_column(SAEnum(MatchStatus), default=MatchStatus.unmatched)
    match_cost: Mapped[Optional[float]]
    created_at: Mapped[datetime] = mapped_column(default=utcnow)
    image: Mapped[Image] = relationship(back_populates="observations")
    mole: Mapped[Optional[Mole]] = relationship(back_populates="observations")


class ContourComparison(Base):        # cached result of overlaying two observations of one mole
    __tablename__ = "contour_comparisons"
    __table_args__ = (UniqueConstraint("from_observation_id", "to_observation_id"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    mole_id: Mapped[int] = mapped_column(ForeignKey("moles.id"), index=True)
    from_observation_id: Mapped[int] = mapped_column(ForeignKey("observations.id"))
    to_observation_id: Mapped[int] = mapped_column(ForeignKey("observations.id"))
    # alignment used (rigid only, never scaled)
    rotation_deg: Mapped[float]
    shift_x_mm: Mapped[float]
    shift_y_mm: Mapped[float]
    # change metrics
    iou: Mapped[float]
    dice: Mapped[float]
    grown_mm2: Mapped[float]
    shrunk_mm2: Mapped[float]
    hausdorff_mm: Mapped[float]
    mean_contour_dist_mm: Mapped[float]
    max_radial_growth_mm: Mapped[float]
    max_growth_angle_deg: Mapped[float]
    shape_change: Mapped[Optional[float]]                    # Fourier-descriptor distance (size-normalised)
    radial_profile_diff: Mapped[Optional[list[Any]]]         # 360 values, mm
    overlay_path: Mapped[Optional[str]] = mapped_column(String(500))    # rendered overlay PNG
    created_at: Mapped[datetime] = mapped_column(default=utcnow)


class Alert(Base):
    __tablename__ = "alerts"
    id: Mapped[int] = mapped_column(primary_key=True)
    mole_id: Mapped[int] = mapped_column(ForeignKey("moles.id"), index=True)
    observation_id: Mapped[int] = mapped_column(ForeignKey("observations.id"))
    kind: Mapped[str] = mapped_column(String(30))            # area_growth | diameter | colour_shift | new_colour | new_mole | outline_change | one_sided_growth
    value: Mapped[float]
    threshold: Mapped[float]
    created_at: Mapped[datetime] = mapped_column(default=utcnow)
    acknowledged: Mapped[bool] = mapped_column(default=False)
    mole: Mapped[Mole] = relationship(back_populates="alerts")
```

Claude Code may refine these models (e.g. add indexes, a `NoiseFloor` table for per-metric σ from repeatability sessions) but must keep them consistent with this plan and add a migration for every change.

---

## 6. Web UI — Next.js (`web/`) + local FastAPI (`moletracker/api/`)

The Next.js app runs locally (`npm run dev`) and talks only to the local FastAPI backend (`http://127.0.0.1:8000`), which runs the pipeline and is the only thing that touches the database and `data/`. Images are served to the browser by the API. Visual style is defined by me in the `ui-frontend-expert` agent's style guide.

Pages:

1. **New session**: choose region → upload photos or a video → show detected moles overlaid (boxes + IDs), allow add/delete/adjust boxes manually.
2. **Review matches**: side-by-side reference vs new, with proposed links; confirm / reassign / mark as new.
3. **Mole timeline**: per mole, aligned normalised crops over time; **outline overlay** of any two dates (pick baseline/previous/any), grown area shaded, shrunk area hatched, 1 mm scale bar; time slider through all outlines; polar plot of radial profiles; charts of diameter, area, IoU vs baseline, max radial growth, ΔE, asymmetry; alerts listed.
4. **Body overview**: list of regions with mole counts, last capture date, active alerts.
5. **Export**: PDF/HTML report of a mole's history to show a dermatologist.

---

## 7. Milestones & acceptance criteria

| # | Milestone | Done when |
|---|---|---|
| M0 | Setup + repo inspection | Project skeleton, config, `compose.yaml` with local PostgreSQL, SQLAlchemy models + initial Alembic migration, `alembic upgrade head` creates the schema; FastAPI app with a health endpoint; empty Next.js app in `web/` that calls it; written summary of the external repo, DiaMole and LKM-SAM (model, weights, licence, input type, code availability, usable or not). |
| M1 | Calibration | Printable marker sheet; on a test photo, mm/px within ±3 % of ruler; colour correction reduces patch error. |
| M2 | Manual tracker | Upload photo, click moles (or draw boxes), Otsu segmentation, all measurements **and contour** stored, raw + normalised crops/masks saved. Usable end-to-end. |
| M3 | Auto detection | Blob baseline + optional repo/YOLO wrapper; detections editable in UI. |
| M4 | SAM segmentation | Box-prompted SAM behind the Segmenter interface; switch via config; compare with Otsu on my images. |
| M5 | Registration + matching | Second session of a region auto-matched; I confirm in UI; ≥ 90 % correct matches on my data. |
| M6 | Change detection + contour overlay | Contours stored per observation; two dates of a mole overlaid and aligned (rigid, no scaling); IoU, grown/shrunk mm², Hausdorff, radial growth per direction computed and cached; repeatability session sets noise floor for all metrics; timelines, overlays and alerts work; report export includes overlays. |
| M7 | Video multi-frame | Detections merged across frames (frame-to-frame homography/tracking), duplicate-free per region. |
| M8 (optional) | Research add-ons | Fine-tune YOLO on iToBoS; experimental risk score clearly labelled "not a diagnosis". |

---

## 8. Configuration (`config.yaml`), example

```yaml
marker:
  dictionary: DICT_4X4_50
  side_mm: 20.0
  min_side_px: 80
  max_angle_deg: 30
colour:
  reference: grey_strip        # or colorchecker24
detector: blob                 # blob | repo | yolo
segmenter: otsu                # otsu | sam | lkm_sam
sam:
  checkpoint: null             # path to SAM / MedSAM / skin-SAM weights
matching:
  gate_mm: 5.0
  w_area: 2.0
  w_colour: 0.2
alerts:
  area_pct: 20
  diameter_mm: 1.0
  deltaE: 8
  iou_min: 0.80               # below this vs baseline (and above noise) -> outline_change
  radial_growth_mm: 0.5       # max growth in any direction -> one_sided_growth
  confirm_sessions: 2         # contour alerts need 2 consecutive sessions unless large
contours:
  n_points: 256
  norm_mm_per_px: 0.05
  max_align_rotation_deg: 15
  max_align_shift_mm: 1.0
  sigma_multiplier: 3
```

---

## 9. Testing

- **Synthetic images:** generate skin-coloured backgrounds with a rendered ArUco marker and dark ellipses of known mm size, colour and position; assert measured diameter within ±0.3 mm, area within ±5 %, position within ±0.5 mm, ΔE recovery after a simulated colour cast.
- **Contour tests:** render a synthetic mole, then a "later" version that grows only on one side and is slightly rotated/shifted; assert the alignment recovers the rotation/shift, IoU and grown/shrunk areas match the known values (±5 %), and max radial growth points in the right direction (±10°). Also assert that a pure re-photograph (no change, new noise) stays under the alert thresholds.
- **Matching tests:** take a synthetic constellation, apply random rotation/translation + noise + missing/new points; assert correct matching.
- **Real-data checks:** repeatability session (same region, 3 photos) → report the coefficient of variation per metric.

---

## 10. Privacy & safety

- Everything local; `data/` in `.gitignore`; optional encryption of the data folder.
- PostgreSQL, the FastAPI backend and the Next.js dev server all bind to `127.0.0.1` only; nothing is exposed to the network.
- DB credentials live in a git-ignored `.env`; only `.env.example` is committed.
- No telemetry, no cloud APIs for images. No cloud deployment for now (see Decisions log).
- UI disclaimer: "Tracks changes only. Not a medical device. Any new, changing, or unusual mole should be checked by a dermatologist."
