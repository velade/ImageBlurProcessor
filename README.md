# ImageBlurProcessor
## 介紹
這是一個超輕量級，易於使用的圖片模糊腳本，由純原生Javascript編寫，未使用任何其他圖像依賴，僅使用了fs模組來保存檔案。同時兼容nwjs/electron等框架（理論上也支援QQNT之類的所有基於node.js+chromium的框架）和原生js。

### 為什麼選擇 ImageBlurProcessor？
我知道大多數人會選擇使用Sharp這類的第三方庫，甚至你去問AI它也會這樣推薦。但在某些情況下，ImageBlurProcessor或許是你唯一的選擇，這也是我編寫它的原因。因為Sharp基於libvips，又或者因為它自身的某些原因，它會要求一定的版本，而如果你要開發一個兼容win7的nwjs程式你就不得不使用舊版的nwjs框架，而這個舊版框架的node.js版本就達不到Sharp的最低要求。而我嘗試尋找替代品，發現並沒有滿足需求的同時又足夠易用的替代品，所以ImageBlurProcessor誕生了。

### 為什麼不使用CSS的模糊功能？
大多數情況下，CSS的模糊可以解決許多的問題，但不是所有情況都適合採用CSS的模糊，尤其是從效能的考量上，CSS的模糊是動態模糊，而且不能直接作用於背景，backdrop-filter雖然可以作用於背景但效能較差。又或者你想要你的窗口有類似Win11那種「毛玻璃」效果，但實際上backdrop-filter無法作用於nwjs之外。這時一個比較好的解決方案是獲取背景圖片將它預先模糊後，在窗口移動時通過變更background-position來達成。這樣的技術被稱為「靜態模糊」，這時你就需要一個能夠預先製作一個模糊後的圖片的方法，ImageBlurProcessor就是為此而設計的。
另一種情況是，你要模糊一些敏感圖片，使用CSS即時模糊實在太容易破解，只需要打開「開發者工具」改一下模糊度就破解了，這時預先模糊的技術就派上用場了，因為徹底的修改了圖像的像素，因此想要恢復幾乎是不可能的。

## 使用方法
1. 引入
    - 通過CommonJS的`const ImageBlurProcessor = require('/path/to/ImageBlurProcessor.js');`引入
    - 刪除腳本結尾的`module.exports = ImageBlurProcessor;`並以普通js方式引入（html的script標籤）
    - 修改結尾的`module.exports = ImageBlurProcessor;`為`export default ImageBlurProcessor;`並以ES Module的方式引入，即`import ImageBlurProcessor from '/path/to/ImageBlurProcessor.js'`
2. 使用
   1. 創建ImageBlurProcessor實例 `new ImageBlurProcessor(inputPath, targetSize, zipRate, isLocalPath)`
      - inputPath 原始圖片的路徑
      - targetSize 目標尺寸：你需要給出一個固定的分辨率，以[width, height]來表示
      - zipRate （可選）壓縮比例：模糊的圖片本身不需要太過高清，因此你可以在這裡設定一個壓縮比例來降低分辨率，範圍是0.01-1，0的話圖片會消失，因此要大於0；1為不壓縮，超過1是強行放大，但無意義。例如設定為0.25則是生成的圖片是目標尺存的1/4
      - isLocalPath （可選）是否為本地圖片：在nwjs或electron這類框架中，你可能會用到相對本機的絕對路徑，例如D:\pictrues\img0.jpg (Windows)或/home/xxx/Pictures/img0.jpg (Linux)等，通過將此參數設定為true，可以讀取本地文件而不是相對應用根目錄的路徑（在純js中可能無效）。或者也可以在inputPath直接加上"file://"前綴，這兩種方式僅可選擇其一。
   2. 調用blurImage 這是一個Promise方法，你需要通過then異步獲取結果，返回結果為bluredBlob這個自定義類，因為要對後續操作進行封装。你如果只是想要blob本身可以通過bluredBlob.blob獲取原始的blob。
      - blurRadius 模糊半徑，不是有些圖形處理使用的sigma，是和CSS一致的像素單位模糊半徑。
      - blobType （可選）返回的blob的MIME類型，默認為"image/webp"，這同時決定了你之後要保存、下載獲取DataURL時的檔案格式。
   3. 調用bluredBlob下的方法來保存到檔案、獲取Base64格式的DataURL，或直接調出下載讓用戶保存到本地。
      - toFile(filePath) 保存到檔案，filePath為完整的圖片路徑，這裡使用fs保存，因此必須在nwjs或electron這類環境中使用，路徑為本地絕對路徑。支援Promise，但不會有任何返回值。
      - toDataUrl() 獲取Base64格式的DataURL，注意是**異步**操作，通過.then獲取結果，不支援`var dataurl = bluredblob.toDataUrl()`這種同步獲取方式。
      - download(filename) 觸發下載（另存為窗口）。在nwjs或electron中可能不會有下載進度的顯示，因此點完保存後不建議立即關閉窗口，可能會導致下載中斷。
        - filename （可選）指定下載檔案的名稱（如果彈出保存窗口，用戶可以保存窗口修改），如未指定則以 原檔案名_blured 的名稱保存，副檔名根據之前指定的blobType自動決定。

  
## 使用示例
```javascript
// 引入CommonJS模快
const ImageBlurProcessor = require(./ImageBlurProcessor.js);

const inputPath = "/path/to/image.jpg";
const targetSize = [1920,1080];
const blurRadius = 30;

// 實例化ImageBlurProcessor類（這裡設定了實際圖片大小為1/4，並且原圖片是本地絕對路徑）
const processor = new ImageBlurProcessor(inputPath, targetSize, 0.25, true);

// 執行圖片模糊 這裡沒有給blobType的參數，因此默認 "image/webp"
processor.blurImage(blurRadius).then((bluredblob) => {
    // 保存到文件
    bluredblob.toFile("/path/to/save/wallpaper.webp");
    // 獲取Data URL
    bluredblob.toDataUrl().then(dataUrl=>{
        console.log(dataUrl); // data:image/webp;base64,Ukl......A=
    });
    // 以filename.webp的檔案名下載
    bluredblob.download("filename.webp");
    // 以默認的名稱 image_blured.webp 下載
    bluredblob.download();
})
```

## 使用建議
建議除非特殊情況，使用默認的image/webp作為輸出格式，這是專為網頁體驗設計的圖片格式，在壓縮率、性能等方面的表現要優於傳統的jpg和png。

可以通過將模糊後的圖片設定為背景，並使用win.move((x,y)=>{})監視並設定backgroud-position來達到Win11那種靜態毛玻璃效果喔（是的如果你仔細觀察，Win11的毛玻璃其實不具備透明度，他是取壁紙進行模糊後設為背景的靜態模糊方案，和Win7的Aero效果可以透視後面的窗口是不同的）
