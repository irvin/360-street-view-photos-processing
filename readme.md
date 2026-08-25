# 處理街景照片流程

## 根據照片的 EXIF 時間寫入 GPX 軌跡座標並上傳

1. 用 insta360 Studio 輸出 360 照片
2. 用 exiftool 根據 gpx 寫入 GPS coord 上去照片

    時間必須要是 GMT，如果用+8 mapillary 會錯
    ```
    geotag_with_gpx origin test.gpx geocoded
    geotag_with_gpx origin test.gpx geocoded +09:00 //如果照片不在本地時區拍攝，就要加上時區參數
    ```
3. 根據 GPS 座標去除重複照片
    ```
    checkimg-latlong-dup geocoded dupe
    ```
4. 移除低於 1kph 以下的圖片移除（行車）
    ```
    checkimg-speed-dupe geocoded dupe 1 --until-stable
    ```
5. 手動檢視並刪除重複的照片
6. 計算並寫入方位角 interpolate
    ```
    calcimg-dir geocoded 180
    ```
7. 上傳 mapillary
    ```
    mapillary_tools process_and_upload geocoded \
        --user_name "irvinfly" \
        --organization_key "314354923595106"
    ```
8. 上傳 Panoramax
    上傳前請確認相片目錄內有 `panoramax.csv`。檔名必須是
    `panoramax.csv`，且至少包含 `file,capture_time`；`capture_time`
    應使用保留拍攝地時區的 RFC 3339 時間，例如
    `2026-08-14T18:06:32+12:00`。這會正確表達拍攝時刻及其原始時區；
    Panoramax 網頁實際如何顯示時間，請參閱後方的「上傳後驗證與
    Panoramax Taiwan 已知行為」。

    ```
    panoramax_cli upload --api-url https://panoramax.osm.tw --split-distance 200 geocoded
    ```

9. 從 geocoded JPG 產生含 `<time>` 與分段 `<trkseg>` 的 GPX 軌跡。
    ```
    jpg-to-gpx geocoded output-segmented.gpx --force
    ```

    批次處理指定 root 第一層的每個資料夾，在各資料夾內產生同名 GPX：
    ```
    jpg-to-gpx --batch photos --force
    ```

    每份 GPX 會遞迴讀取該資料夾下所有 JPG，並依照片所在資料夾建立具名
    `<trk>`；同一個 `<trk>` 內仍會依距離、時間差或缺少 GPS 拆成多個
    `<trkseg>`。

## MIO 影片轉檔與 Panoramax 時間

### 從 MIO 影片與 NMEA 轉出 JPEG

MIO 的一般轉檔模式會在輸出目錄自動產生 `panoramax.csv`，不需要另外
加參數。CSV 會把轉檔時由 NMEA／`--offset` 計算出的當地拍攝時間，以
含時區的 RFC 3339 格式寫入，例如 `2026-08-14T18:06:32+12:00`。
這是照片原始拍攝時間的正確表達；`--offset` 是拍攝地的 UTC 時區差，
應依拍攝當下所在地的時區規則設定。

例如：

```bash
cd /Users/Irvin/Coding/360-street-view-photos-processing/mio-dashcam-mp4-to-jpg

node batch-extract.js \
  --video ./MIO/F \
  --nmea ./MIO/NMEA \
  --out ./MIO_out \
  --offset +12:00 \
  --jpeg-quality 3 \
  --frame-offset 7 \
  --crop 2560x1355 \
  --make Mio \
  --model "MiVu 868W"
```

輸出會放在 `./MIO_out/<影片主檔名>/`，其中包含 JPEG 與
`panoramax.csv`。如果只需要先建立 CSV 清單、不抽取或修改 JPEG，可加
上 `--panoramax-csv-only`；這種模式產生的是預計輸出清單，必須在相片
實際存在後才能直接上傳。

```bash
node batch-extract.js \
  --video ./MIO/F \
  --nmea ./MIO/NMEA \
  --out ./MIO_out \
  --offset +12:00 \
  --panoramax-csv-only
```

### 從既有 JPEG 補產 `panoramax.csv`

如果 JPEG 已經存在但目錄內沒有 CSV，可使用
`photos-de-dupe-scripts/make_panoramax_csv.js`。工具會遞迴掃描指定目錄，
在每個含有 `.jpg`／`.jpeg` 的資料夾寫入一份同層的 `panoramax.csv`。

```bash
cd /Users/Irvin/Coding/360-street-view-photos-processing

node photos-de-dupe-scripts/make_panoramax_csv.js \
  "/Users/Irvin/Downloads/360 photos/wip/20260814/MIO"
```

這個工具從照片 EXIF 的 `DateTimeOriginal` 讀取牆上時間，預設使用
`OffsetTimeOriginal` 作為時區；如果照片沒有 EXIF 時區，可以明確指定：

```bash
node photos-de-dupe-scripts/make_panoramax_csv.js \
  "/Users/Irvin/Downloads/360 photos/wip/20260814/MIO" \
  --timezone +12:00
```

`--timezone` 只是在輸出中指定時區 offset，不會把錯誤的牆上時間重新換算。
工具也不從檔名推算時間；若照片缺少 EXIF 拍攝時間，該照片會被略過。

### 上傳前檢查

Panoramax CLI 只會自動讀取放在相片同一目錄、且檔名完全是
`panoramax.csv` 的外部 metadata。CSV 的 `file` 必須是相片的精確檔名，
`capture_time` 應是帶時區的 RFC 3339 時間；GPS 座標仍須存在照片 EXIF
中，CSV 不會替照片補 GPS。不要為了讓網頁顯示當地時間而移除 `+12:00`
等 offset，否則會失去原始拍攝時間的時區語意。

確認 CSV 產生後，從包含 JPEG 與 CSV 的目錄執行上傳：

```bash
panoramax_cli upload \
  --api-url https://panoramax.osm.tw \
  --split-distance 200 \
  ./MIO_out/EMER260816-094812F
```

若使用既有相片目錄，則將最後的路徑替換為該目錄，例如 `./MIO`。

### 上傳後驗證與 Panoramax Taiwan 已知行為

Panoramax 會將用於排序與 STAC 查詢的 `datetime` 正規化為 UTC，這是正常
行為；但 API 的 `datetimetz` 應保留原始拍攝時間的時區，供 web viewer
顯示拍攝地當地時間。

2026-08-25 實測 `https://panoramax.osm.tw`：即使 CSV、EXIF 與 XMP 都含有
`2026-08-14T18:06:32+12:00`，該 instance 回傳的 `datetime` 和
`datetimetz` 都是 `2026-08-14T06:06:32+00:00`；原始 `+12:00` 仍存在
`properties.exif`。因此 viewer 顯示 GMT+00 是 server API 資料的結果，
不是轉檔程式或 `panoramax.csv` 造成的問題。

Panoramax 官方 server 文件沒有「啟用拍攝地時區」的設定，並且要求
PostgreSQL 資料庫使用 UTC；不要將資料庫或主機時區改成拍攝地時區。
應確認 API 與 web viewer 版本，並以 `datetimetz` 的 API 回應作為排查依據。

```bash
# 先由 upload set 找出本次建立的 collection ID
curl -fsSL \
  'https://panoramax.osm.tw/api/upload_sets/<upload-set-id>' | jq \
  '.associated_collections[] | {id, title, ready}'

# 再檢查第一張照片的時間欄位與原始 EXIF 時區
curl -fsSL \
  'https://panoramax.osm.tw/api/collections/<collection-id>/items?limit=1' | jq \
  '.features[0].properties | {
    datetime,
    datetimetz,
    original: .exif["Xmp.exif.DateTimeOriginal"],
    offset: .exif["Exif.Photo.OffsetTimeOriginal"]
  }'
```

若 `original`／`offset` 含有拍攝地時區，但 `datetimetz` 仍是 `+00:00`，
請將該 API 回應連同 API 與 web viewer 版本提交給 Panoramax upstream；
這是 server 端應修正的時區保存問題。

## 下載 Mapillary 圖片

1. 截取特定使用者的 sequences 清單
    ```
    source mapillary_env/bin/activate && python3 find_sequences_of_user.py irvinfly -f regular
    ```
2. 批次下載該清單下的照片
    ```
    source mapillary_env/bin/activate && python3 batch_downloader.py sequences_irvinfly.txt
    ```
