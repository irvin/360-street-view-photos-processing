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
    checkimg-speed-dupe geocoded dupe 1
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

## 下載 Mapillary 圖片

1. 截取特定使用者的 sequences 清單
    ```
    source mapillary_env/bin/activate && python3 find_sequences_of_user.py irvinfly -f regular
    ```
2. 批次下載該清單下的照片
    ```
    source mapillary_env/bin/activate && python3 batch_downloader.py sequences_irvinfly.txt
    ```
