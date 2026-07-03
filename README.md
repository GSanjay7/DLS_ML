# DLS_ML
dls-microplastic-moe/
├── README.md, LICENSE (MIT), requirements.txt, pyproject.toml
├── .gitignore
├── dlsmp/                        # sanitised Python package (10 modules)
├── data/
│   ├── processed/                # 27-measurement feature matrix, distributions, engineered features
│   ├── synthetic/                # 450-sample synthetic_augmented.npz
│   └── external/                 # Wei and Chua (2026) processed table
├── models/                       # model_moe_primary.joblib, model_moe_shape.joblib (retrained under sanitised package)
├── figures/                      # 18 PNG figures with clean names (fig1_*, fig2_*, ..., figS_*)
├── scripts/                      # make_figures.py
└── tests/                        # 9 sanity tests, all passing
