# AIRMap_AI-Generated-Radio-Maps-for-Wireless-Digital-Twins-Notes
基於論文之重點整理

# AIRMap — AI-Generated Radio Maps for Wireless Digital Twins  
閱讀與查核筆記（面試用）

作者：Luo Wen-Jun（羅文均）  
論文來源：arXiv:2511.05522v1  
GitHub 紀錄目的：
- 紀錄自己研讀論文的規劃方式與執行歷程
- 梳理每個查核點的關鍵知識與待解決問題
- 作為面試簡報時的「後台筆記」，用來佐證自己的理解與觀點

---

## 0. 研讀規劃與時間預估

### 0.1 整體時間預估
| Phase 1 | 快速通讀  | 1.0 hr |
| Phase 2 | 尋找答案 | 2hr |
| Phase 3 | 整理面試簡報 & GitHub 筆記，| 1.5 hr |

## 1. 查核點 #1 — 問題定義、重要性與困難點（Abstract + Intro）


### 1.1 閱讀內容摘要(第 1 題：作者要解決什麼問題？為什麼重要？為什麼困難？)

- **問題定義**  
  - 在無線網路的 Digital Twin 中，需要一個既「準確」又「超低延遲」的 channel 模型，用來即時生成 **radio map**（例如 path gain map），支援 O-RAN dApps 等 sub-10 ms 的控制應用。  
  - 傳統 Measurement-based / Statistical / Ray-tracing 模型，各自都有 **準確 vs 延遲 vs 成本** 的 trade-off，難以同時滿足 real-time digital twin 的需求。  
  - AIRMap 嘗試用單一 U-Net 模型，从 2D elevation map 直接推估 radio map。

  參考頁面：  
  - Abstract 對整體問題、方法與結果的總結 (p.1, 約 L18–35)  
  - Digital twin 的角色與重要性說明 (p.1, 約 L37–45)  

- **為什麼重要？**  
  - O-RAN/RAN 中不同應用對延遲有不同要求：  
    - rApps：> 1 s  
    - xApps：10–1000 ms  
    - dApps：< 10 ms  
  - 若 channel 模型本身計算就需要秒級甚至更久，就無法支撐 near real-time / real-time 的決策與閉迴路控制。  
  - 未來 B5G/6G 推 Digital Twin、NTN/LEO、AI-native RAN，都需要這種高速 channel 模型。

  參考頁面：  
  - O-RAN real-time 等級與 dApps 說明 (p.2, 約 L30–37, L46–55)

- **為什麼困難？**  
  - Measurement-based：  
    - 成本高，只能覆蓋有限範圍；環境變動（新建物、車流、人流）後容易過期。  
  - Statistical model：  
    - 假設簡化，只能描述「平均」行為，對特定城市、特定場景的細節（如遮蔽、街谷效應）掌握有限。  
  - Ray tracing：  
    - 雖然理論上很準，但：  
      - 需要精細 3D 模型與材料參數  
      - 計算量巨大，即使 GPU 也常達秒級甚至更久  
      - 難以頻繁重算、難以隨環境變化更新  

  參考頁面：  
  - 傳統三類 channel 模型與缺點 (p.2, 約 L1–18)

### 1.2 我學到的知識（Knowledge）

- Digital Twin 在無線通訊中的主要用途：不只是離線規劃，也要能做 **real-time decision support**（例如 beam selection、resource allocation、handover）。  
- 現有 channel 模型在 **精度、速度與實務維運成本** 三者之間有明顯衝突。  
- O-RAN 架構中，**dApps 的 <10 ms 延遲要求** 直接限制了通道建模方法能用什麼工具（ray tracing 幾乎被淘汰）。  

### 1.3 閱讀時產生的問題（Questions）

- 在實際 5G/6G 網路中，數位孿生會多常重新更新 radio map？是秒級、分鐘級，還是以事件驅動？  
- 若環境變動頻率很高（例如車流密度改變），AIRMap 這種 U-Net 模型需要多久更新一次，才不會失真？  
- O-RAN 中的 dApps 實際現在有哪些 use case 已經上線？這些應用對 channel 模型的精度要求到什麼程度？

---

## 2. 查核點 #2 — 現有解法與 Ray Tracing 限制（Measurement & RT 分析）

**對應老師第 2 題：現有解決方法有什麼缺點？**  
**主要閱讀範圍：** p.2–p.5

### 2.1 閱讀內容摘要

- **Measurement Campaign 設計**  
  - 頻率約 933 MHz，場景在 Northeastern 校園附近。  
  - TX：頂樓固定位置；RX：沿路軌跡移動量測 CIR，從中計算 path gain。  
  - 在 anechoic chamber 事先校正所有硬體增益與損耗。  
  - Path gain 根據 CIR 的能量加總再 log（對應論文中的數學式）。  
  - 參考頁：p.4, 關於 measurement setup 與 path gain 計算。  

- **Empirical model 與 Ray Tracing 的問題**  
  - 經驗路損公式（單純用距離 + path loss exponent）無法很好 fit 實測（Fig. 4）。  
  - Ray tracing 即使用「高深度 + diffraction on」設定，與實測仍存在明顯誤差。  
  - 作者進一步掃描 Sionna RT 參數：  
    - path depth 增加 → runtime 大幅增加，RMSE 改善有限（存在 sweet spot）。  
    - diffraction on vs off → 有 diffraction 時精度較好，但計算更貴。  
  - 最後決定使用 diffraction on + depth=10 作為資料集生成的折衷設定，每張 radio map 約 30 秒。  

### 2.2 我學到的知識（Knowledge）

- 實務上要做大量 site-specific 數據集時，**ray tracing 的參數選擇需要做「系統性掃描」**，不只是隨便選一組。  
- 即使是「高設定」的 ray tracing，也不會等於實測；其實它本身就是一種近似，model mismatch 仍然存在。  
- 作者非常強調 **runtime vs RMSE 的 trade-off**，這個概念對未來做任何模擬工具設計都很重要。

### 2.3 閱讀時產生的問題（Questions）

- Sionna RT 所使用的材料屬性與實際建築材料有多接近？是否有特別校正？  
- 若換成 mmWave 頻段，繞射與穿透的建模誤差會不會更嚴重？  
- Ray tracing 的「深度=10」這個 sweet spot 是否會隨環境（dense urban vs suburban）改變，需要場景特定調整？

---

## 3. 查核點 #3 — Dataset Pipeline 與 System Model（3D → 2D Elevation → Radio Map）

**對應老師第 3 題：用系統架構圖說明 I/P 與 O/P**  
**主要閱讀範圍：** p.6–p.7（Fig. 8 等）

### 3.1 閱讀內容摘要

- **Dataset 生成流程（System Model 的 Offline 部分）**  
  1. 從 BostonTwin 3D 城市模型中選擇 TX/RX 位置（合法位置）。  
  2. 使用 Sionna RT 以預設參數（diffraction on, depth=10）產生真值 radio map（path-gain map）。  
  3. 同時將該區域的 3D 資料轉成 2D **elevation map**（地形＋建物高度投影）。  
  4. 為每個 TX 生成多個 random coverage scale（500 m–3 km）與旋轉／翻轉的 augmentation。  
  5. 建立 (elevation map, radio map) pair dataset，再划分 train/val/test。  

- **Model I/P 與 O/P**  
  - I/P：單通道 2D elevation map，TX 固定置中。  
  - O/P：與輸入同尺寸的 path-gain radio map（每個 pixel 一個 path gain 值）。  
  - 同一個 U-Net 架構可以支援不同 coverage scale，只要輸入尺寸固定。  

### 3.2 我學到的知識（Knowledge）

- **「把 TX 放在圖片中心」** 是一個聰明的設計：  
  - 不需要額外編碼 TX 座標，模型自然把中心當作「發射源」。  
- 將 3D 城市模型降維到 2D elevation map，其實是一種「保留最有用資訊」的壓縮：  
  - 對 path gain 來說，建物高度與基本地形足以描述大尺度遮蔽效應。  
- 資料增強（旋轉、翻轉）在空間問題尤其重要，可以提升模型對各種方位與場景的泛化能力。

### 3.3 閱讀時產生的問題（Questions）

- 如果未來要支持多個 TX（例如 multi-cell deployment），是否需要改成多通道輸入（例如：每個 TX 一個 channel）？  
- elevation map 是否有考慮到「材質」差異？目前看起來只考慮高度，是否會導致某些 case（例如玻璃 vs 混凝土）無法被區分？  
- 是否可以直接使用現成的 GIS / DEM 資料（例如政府開放資料）當 elevation map，而不必依賴高精度 3D 模型？

---

## 4. 查核點 #4 — AIRMap U-Net 架構與訓練策略（Proposed Method）

**對應老師第 4 題：作者提出什麼概念？用哪些效能指標證明？**  
**主要閱讀範圍：** p.7–p.8

### 4.1 閱讀內容摘要

- **U-Net 架構特色**  
  - Encoder：多層 conv + downsampling，搭配殘差單元與 Atrous Spatial Pyramid Pooling (ASPP) 擴大 receptive field。  
  - Decoder：upsampling + skip connections，重建細節。  
  - Output：單通道 path-gain map（與輸入同尺寸）。  

- **Loss 與訓練細節（高層次理解）**  
  - 使用 MSE 或類似 regression loss 作為主 loss，目標是最小化預測 path gain 與真值 RT map 的差異。  
  - 訓練時大量利用 ray tracing 生成的 simulation data。  
  - 之後再透過實測 data 做 fine-tuning（詳見下個查核點）。  

- **效能指標**  
  - Path gain prediction RMSE < 5 dB，全測試集的 median error ≈ 4.43%。  
  - 推論時間（NVIDIA L40S）約 4.2 ms，比 GPU 加速 ray tracing 快 7000×。  

### 4.2 我學到的知識（Knowledge）

- 影像式架構（U-Net）非常適合 radio map 這種 2D spatial 問題，因為 path gain 與環境在空間上的相關性本質上就是「圖像」。  
- 加入 ASPP 等多尺度模組可以讓模型同時看到「局部建物形狀」與「大尺度城市結構」。  
- 在通道模型的 context 中，RMSE < 5 dB 已經是一個相當有競爭力的數字，尤其是在 coverage 全域上。

### 4.3 閱讀時產生的問題（Questions）

- 模型是否有專門對「LOS vs NLOS」做分類或特別設計？  
- 如果要讓模型同時預測其他參數（例如 delay spread、angular spread），架構是否需要變為多輸出 channel？  
- 模型大小（約 37M params）在 edge device 上是否可接受？有沒有考慮過 model compression / pruning？

---

## 5. 查核點 #5 — Calibration 與實驗結果（Simulation/Experimental Results）

**對應老師第 5 題：作者如何證明方法已成功解決問題？**  
**主要閱讀範圍：** p.8–p.10（Calibration, eCDF, Fig. 10–13）

### 5.1 閱讀內容摘要

- **Calibration（實測校正）流程**  
  - 在真實網路中收集有限的實測點（含 RX 位置與 path gain）。  
  - 透過 mapping，把這些測量點投影到 elevation map pixel 上。  
  - 使用加權 loss：對實測點給較高權重，以縮小 simulation-to-reality gap。  
  - 使用 20% 的實測資料作為 calibration training set，剩餘 80% 做測試；採地理區分 split，避免「空間重疊」造成過擬合。  

- **Calibration 效果**  
  - 純 ray tracing：中位數誤差大於 50%。  
  - 未校正 U-Net：中位數誤差約 8–9%。  
  - 校正後 U-Net：中位數誤差約 10%，但對系統層指標的表現更穩定。  
  - 作者重複 100 次實驗（不同 train/test split）來確保結果穩健。  

- **視覺比較與 eCDF**  
  - Fig. 10：U-Net 預測 radio map 與 ray tracing 真值在不同場景下高度相似。  
  - Fig. 11–13：透過 error eCDF 的方式呈現誤差分佈，而非只看單一 RMSE，顯示模型在大多數點上誤差都落在可接受範圍內。

### 5.2 我學到的知識（Knowledge）

- **Calibration 不只是「再訓練一次」**，而是要設計合適的 train/test split（地理區分）與權重設計，避免只對訓練筆做 overfit。  
- 即使 path gain prediction 的誤差數字看起來有 8–10%，系統層指標（spectral efficiency / BLER）仍可能非常接近實測，表示有些誤差在系統層是「不敏感」的。  
- 以 eCDF 來表達誤差分佈，比單一 RMSE 更有資訊量（可以看出 tail behavior）。

### 5.3 閱讀時產生的問題（Questions）

- 為什麼 calibration 之後的數字在某些統計上看起來「略差」（例如 10% vs 8.7%），但系統層指標反而更好？  
- 加權 loss 的設計是否有「最佳權重比」？作者是不是有做 sensitivity analysis？  
- 若實測資料的數量非常少（例如 < 5%），這套 calibration 方法在實務上還可行嗎？

---

## 6. 查核點 #6 — 與 Sionna SYS & Colosseum 整合（系統層驗證）

**對應老師第 3 + 5 題：System Model 的後半段與系統驗證**  
**主要閱讀範圍：** p.10–p.11（Sionna SYS, Colosseum, OAI 實驗）

### 6.1 閱讀內容摘要

- **Sionna SYS 整合**  
  - Pipeline：  
    1. Elevation map →  
    2. AIRMap (U-Net) 產生 radio map →  
    3. Sionna SYS 做 link-level/system-level simulation（Shannon capacity, spectral efficiency, BLER）。  
  - 結果：  
    - 雖然 radio map 有誤差，容量與 spectral efficiency 的統計分佈幾乎與 measurement-based channel 一致。  

- **Colosseum + MCHEM 整合**  
  - 實作流程：  
    1. 在地圖上選擇 UE/BS 位置 →  
    2. 抽出對應 elevation map patch（< 20 ms）→  
    3. 用 U-Net 推論 radio map（≈ 4 ms）→  
    4. 通過 MQTT 把 channel taps 傳給 Massive Channel Emulator (MCHEM) →  
    5. 在 OAI 上建立實際通訊連線。  
  - 比較不同 channel source：  
    - Ray tracing / 未校正 U-Net：在特定實驗設定下無法穩定建立連線。  
    - 校正後 U-Net：RSRP 分佈與實測最接近，RMSE ≈ 8.86 dB，連線穩定。  

### 6.2 我學到的知識（Knowledge）

- AIRMap 不只是「離線預測工具」，而是已經被整合到 **真實測試床（Colosseum + OAI）** 裡，證明其實用性。  
- 在實際 RAN 系統中，「能不能穩定建立連線」比單純的 path gain RMSE 更重要。  
- 系統層指標（capacity, BLER, RSRP）可以作為驗證 channel 模型的重要依據，補足「純數值 RMSE」的盲點。

### 6.3 閱讀時產生的問題（Questions）

- 若要將 AIRMap 用於商用網路，系統整合會遇到哪些工程瓶頸（例如：延遲、部署方式、資料存取權限）？  
- 在 Colosseum 實驗中，若引入 mobility（UE 移動），radio map 是預先一次性算好，還是可以隨 UE 路徑逐點快速更新？  
- 對 O-RAN 架構而言，AIRMap 最適合部署在 non-RT RIC、near-RT RIC，還是作為一個獨立 service？

---

## 7. 查核點 #7 — 個人延伸題目與未來規劃（對應老師第 6、7 題）

**對應老師第 6 題：如何研讀？用什麼 AI 工具？**  
**對應老師第 7 題：延伸興趣與可能解法？**

### 7.1 我如何研讀這篇論文？

- **流程規劃**  
  - 快速閱讀了解論文主軸  
  - 針對問題尋找對應答案。  
  - 整理這筆記與簡報。  

- **使用的 AI 工具與方式**  
  - 使用 ChatGPT（wenj）協助：  
    - 先做 page-by-page 摘要，整理出各頁重點與 p.X, Lm–Ln 索引。  
    - 確認自己對 U-Net 架構、Dataset pipeline、Calibration 流程的理解是否與原文一致。  
    - 幫忙從論文內容映射到老師要求的 7 題框架。  
  - 使用 PDF viewer 標註：  
    - 對應關鍵圖（Fig. 4, 8, 10–13, 15–17）做註解。  
    - 記錄每一查核點對應的頁面區段。  

### 7.2 基於本論文，我有興趣的議題與可能解法

以下是我目前初步有興趣的方向：

1.**題目 C：Digital Twin 的「自我校正機制」**  
   - 研究 online/continual learning，用少量 live measurement 持續修正 AIRMap 模型。  
   - 探索如何在不中斷服務的情況下做模型更新（例如在 non-RT 平面上輪替模型）。

2. **題目 B：擴展 AIRMap 到 NTN/LEO 通訊**  
   - 將地面 elevation map + LEO 軌道資訊作為多通道輸入，預測時間變化的 radio map。  
   - 探討多普勒、hand-over 頻率對 digital twin 模型的影響。 

3. **題目 A：AIRMap + O-RAN RIC 的即時 beam/resource control**  
   - 利用 AIRMap 提供的快速 radio map，作為 state，搭配 RL/DRL（如 DQN、PPO）設計 beam selection 或 power control 的策略。  
   - 工具：Sionna SYS + PyTorch RL library。  
 

  
