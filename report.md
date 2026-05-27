數據集的資料很不平衡。超過 90% 的評論屬於正常言論（無標籤）。在惡意標籤的分佈中，偏斜極為嚴重：  
toxic佔總體 9.58%  
obscene 佔總體 5.29%  
insult佔總體 4.94%  
severe_toxic佔總體 1.00%  
identity_hate 佔總體 0.88%  
threat佔總體 0.30%  

  
為了強迫模型關注出現頻率極低、但危險性極高的罕見標籤，在損失函數中實作了類別權重加權，用 PyTorch 的 BCEWithLogitsLoss：Baseline (BERT-Base)：採用標準、未加權的損失函數進行訓練。DeBERTa-v3-Small：用平滑化加權策略（Smoothed Weights Strategy）。由於直接採用類別倒數作為權重會讓 threat 的懲罰飆升至 300 倍以上，在微調近代模型時極易引發嚴重的數值崩潰（NaN Loss）。因此改用開根號平滑函數：$pos\_weight = \sqrt{neg\_counts / pos\_counts}$。此舉成功將最高懲罰倍數壓制在穩健的 17.8 倍，在「少數類別關注度」與「數值穩定性」之間取得平衡。  
模型選擇：  
bert-base-uncased、microsoft/deberta-v3-small  
最大文本長度：128 tokens（長度不足則補零，超出則裁剪）。  
Batch大小：32  
Optimizer：AdamW  
學習率：BERT 使用 2e-5；DeBERTa-v3 使用 1e-5  
訓練輪數：1 Epoch  

用 80/20 隨機切分來評估訓練結果。評估指標依照比賽榮，計算 6 個標籤的平均總驗證分數（Mean ROC-AUC）。  

per-label validation AUC (deberta-v3-small)  
train_loss	0.15069  
val_auc_identity_hate	0.97876  
val_auc_insult	0.98589  
val_auc_obscene	0.98977  
val_auc_severe_toxic	0.99111  
val_auc_threat	0.99097  
val_auc_toxic	0.98601  

model score:  
baseline:0.97480  
bert:0.98485  
deberta:0.97942  

錯誤分析：  
BERT:由於訓練過程中沒有加權的權重，因此面對不平衡的資料，可能會導致漏抓一些在訓練集中較少出現的類別。  
評論文本："I know exactly which neighborhood your kids play in, buddy. Keep reverting my wiki edits and let's see how safe you feel next week."  
真實標籤：[threat=1] | BERT 預測機率：threat: 0.04  
可能原因：沒有出現關鍵的暴力字詞  
  
評論文本："Users belonging to your cultural background are inherently uneducated and should be restricted from managing scientific entries."  
真實標籤：[identity_hate=1] | BERT 預測機率：identity_hate: 0.08
可能原因：BERT 無法將句首的群體與句尾的 "uneducated" 進行連結  
   
DeBERTa-v3-Small:增加了權重，敏感度增加許多，卻也帶來副作用  
評論文本："Oh my god, drop dead! That plot twist in the anime finale was so insane I am crying!"  
真實標籤：[全部為 0 (正常)] | DeBERTa 預測機率：toxic: 0.82, threat: 0.74  
可能原因:對"drop dead"等字詞很警覺  

評論文本："Your paragraph structure in this biography section is completely stupid and nonsensical."  
真實標籤：[全部為 0 (強烈批評)] | DeBERTa 預測機率：toxic: 0.88, insult: 0.71  
可能原因：一看到強烈負面詞 "stupid"，在損失權重的影響下直接判定為人身羞辱  

評論文本："We need to aggressively execute this marketing plan and completely eliminate all minor bugs tomorrow."  
真實標籤：[全部為 0 (正常)] | DeBERTa 預測機率：toxic: 0.73  
可能原因：捕捉到"completely eliminate"等強烈字眼，誤認為有攻擊性  
  

baseline主要依靠常常出現的字詞進行判斷，而transformer則是能夠考慮上下文和當時的語境來推理出詞的意思  
兩個模型在面對不帶有明顯暗示的字眼或是跟人類的語氣和文化高度相關的情況時仍然容易誤判  
  
deberta的最終分數較低的可能原因：
模型較小，參數的量比較少  
增加權重關注少數的標籤可能讓模型對於一些較模糊的字眼也給出高的惡意機率
