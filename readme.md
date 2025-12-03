# 處理街景照片流程

## 根據照片的 EXIF 時間寫入 GPX 軌跡座標並上傳

1. 用 insta360 Studio 輸出 360 照片
2. 用 exiftool 根據 gpx 寫入 GPS coord 上去照片

    時間必須要是 GMT，如果用+8 mapillary 會錯
    ```
    geotag_with_gpx origin test.gpx geocoded
    ```
3. 根據 GPS 座標去除重複照片
    ```
    checkimg-latlong-dup geocoded dupe
    ```
4. 手動檢視並刪除重複的照片
5. 計算並寫入方位角 interpolate
    ```
    calcimg-dir geocoded 180
    ```
6. 上傳 mapillary
    ```
    mapillary_tools process_and_upload geocoded \
        --user_name "irvinfly" \
        --organization_key "314354923595106"
    ```
7. 上傳 Panoramax
    ```
    panoramax_cli upload --api-url https://panoramax.liswu.me --split-distance 200 geocoded
    ```

8. 從當前目錄下的所有 JPG 圖片中提取 GPS 資訊，並生成 output.gpx 檔案。
    ```
    exiftool -p "/Users/Irvin/Coding/360-street-view-photos-processing/gpx.fmt" -ee -d %Y-%m-%dT%H:%M:%SZ -fileOrder gpsdatetime geocoded/*.jpg > output.gpx
    ```

## 下載 Mapillary 圖片

1. 截取特定使用者的 sequences 清單
    ```
    source mapillary_env/bin/activate && python3 find_sequences_of_user.py irvinfly -f regular
    ```
2. 批次下載該清單下的照片
    ```
    source mapillary_env/bin/activate && python3 batch_downloader.py sequences_irvinfly.txt
    ```
