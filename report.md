數據集的資料很不平衡。超過 90% 的評論屬於正常言論（無標籤）。在惡意標籤的分佈中，偏斜極為嚴重：  
toxic 與 obscene：最常見的惡意標籤（分別約佔 10% 與 5%）。  
threat 與 identity_hate：極其稀少（分別僅佔約 0.3% 與 0.5%）。  

為了強迫模型關注出現頻率極低、但危險性極高的罕見標籤，在損失函數中實作了類別權重加權，用 PyTorch 的 BCEWithLogitsLoss(pos_weight=...)：Baseline (BERT-Base)：採用標準、未加權的二元交叉熵（BCE）損失函數進行訓練。DeBERTa-v3-Small：實作了平滑化加權策略（Smoothed Weights Strategy）。由於直接採用類別倒數作為權重會讓 threat 的懲罰飆升至 300 倍以上，在微調近代模型時極易引發嚴重的數值雪崩（NaN Loss）。因此，本實驗改用開根號平滑函數：$pos\_weight = \sqrt{neg\_counts / pos\_counts}$。此舉成功將最高懲罰倍數壓制在穩健的 17.8 倍，在「少數類別關注度」與「數值穩定性」之間取得完美平衡。
