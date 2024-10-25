# 程序鏈接

## 靜態鏈接

## 動態鏈接

動態鏈接主要是在程序初始化時或者程序執行的過程中解析變量或者函數的引用。ELF文件中某些節區以及頭部元素就與動態鏈接有關。動態鏈接的模型由操作系統定義並實現。

### Dynamic Linker

動態鏈接器可以用來幫助加載應用所需要的庫並解析庫所導出的動態符號（函數和全局變量）。

當使用動態鏈接來構造程序時，鏈接編輯器會在可執行文件的程序頭部添加一個 PT_INTERP 類型的元素，以便於告訴系統將動態鏈接器作爲程序解釋器來調用。

> 需要注意的是，對於系統提供的動態鏈接器，不同的系統會不同。

可執行程序和動態鏈接器會合作起來爲程序創建進程鏡像，具體的細節如下：

1. 將可執行文件的內存段添加到進程鏡像中。
2. 將共享目標文件的內存段添加到進程鏡像中。
3. 爲可執行文件以及共享目標文件進行重定位。
4. 如果傳遞給了動態鏈接器一個文件描述符的話，就將其關閉。
5. 將控制權傳遞給程序。這讓我們感覺起來就好像程序直接從可執行文件處拿到了執行權限。

鏈接編輯器同樣也創建了各種各樣的數據來協助動態鏈接器來處理可執行文件和共享目標文件，例如

- 類型爲SHT_DYNAMIC的節.dynamic包含了各種各樣的數據，在這個節的開始處包含了其它動態鏈接的信息。
- 類型爲SHT_HASH的節.hash包含了符號哈希表。
- 類型爲SHT_PROGBITS的節.got以及.plt包含了兩個獨立的表：全局偏移表，過程鏈接表。程序會利用過程鏈接表來處理地址獨立代碼。

因爲所有的UNIX System V都會從一個共享目標文件中導入基本的系統服務，動態鏈接器會參與到每一個TIS ELF-conforming的程序執行過程中。

正如程序加載中所說的，共享目標文件中可能會佔用與程序頭部中記錄的不一樣的虛擬地址。動態鏈接器會在程序拿到控制權前，重定位內存鏡像，更新絕對地址。如果共享目標文件確實加載到了其在程序頭部中指定的地址的話，那麼那些絕對地址的值就會是正確的。但通常情況下，這種情況不會發生。

如果進程的環境中有名叫 LD_BIND_NOW 的非空值，那麼動態連接器在把權限傳遞給程序時，會執行所有的重定位。例如，所有的如下環境變量的取值都會指定這一行爲。

- LD_BIND_NOW = 1
- LD_BIND_NOW = on
- LD_BIND_NOW = off

否則，LD_BIND_NOW 要麼不存在於當前的進程環境中，要麼具有一個非空值。動態鏈接器可以延遲解析過程鏈接表的入口，這其實就是plt表的延遲綁定，即當程序需要使用某個符號時，再進行地址解析，這可以減少符號解析以及重定位的負載。


### Function Address

可執行文件中的函數的地址引用和共享目標中與其相關的引用可能並不會被解析爲一個值。共享目標文件中對應的引用將會被動態鏈接器解析到函數本身對應的虛擬地址處。可執行文件中對應的引用（來自於共享目標文件）將會被鏈接編輯器解析爲過程鏈接表中對應函數的項中的地址。

爲了允許不同的函數地址可以按照期望進行工作，如果一個可執行文件引用了一個定義在共享目標文件中的函數，那麼鏈接編輯器就會把相應函數的過程鏈接表項放到與它關聯的符號表表項中。動態鏈接器會對這種符號表項進行特殊的處理。如果動態鏈接器在尋找一個符號，並且遇到了一個符號表項在可執行文件中的符號，那麼它會遵循如下的規則：

1. 如果符號表項的`st_shndx` 不是`SHN_UNDEF ` ，動態鏈接器就會找到這個符號的定義，並且使用它的st_value來作爲符號的地址。
2. 如果`st_shndx` 是`SHN_UNDEF` 並且符號的類型是`STT_FUNC` ，而且`st_value` 成員不是0，動態鏈接器就會把這個表項視爲特殊的，並且使用`st_value` 的值作爲符號的地址。
3. 否則，動態鏈接器就會認爲在可執行文件中的符號是未定義的，然後繼續處理。

一些重定位與過程鏈接表的表項相關。這些表項被用於直接函數調用，而不是引用函數地址。這些重定位並不會按照上面的方式進行處理，因爲動態鏈接器必須不能重定向過程鏈接表項並使其指向它們本身。

### Shared Object Dependencies

當鏈接編輯器在處理一個歸檔庫的時候，它會提取出庫成員並且把它們拷貝到輸出目標文件中。這種靜態鏈接的操作在執行過程中是不需要動態連接器參與的。共享目標文件同時也提供了服務，動態鏈接器必須將合適的共享目標文件attach到進程鏡像中，以便於執行。因此，可執行文件以及共享目標文件會專門描述他們的依賴關係。

當一個動態鏈接器爲一個目標文件創建內存段時，在DT_NEEDED項中描述的依賴給出了需要什麼依賴文件來支持程序的服務。通過不斷地連接被引用的共享目標文件（即使一個共享目標文件被引用多次，它最後也只會被動態鏈接器連接一次）及它們的依賴，動態鏈接器建立了一個完全的進程鏡像。當解析符號引用時，動態鏈接器會使用BFS（廣度優先搜索）來檢查符號表。也就是說，首先，它會檢查可執行文件本身的符號表，然後纔會按照順序檢查DT_NEEDED入口中的符號表，然後纔會繼續查看下一次依賴，依次類推。共享目標文件必須可以被程序讀取，其它權限不一定需要。

依賴列表中的名字要麼是DT_SONAME中的字符串，要麼是用於構建對應目標文件的共享目標文件的路徑名。例如，如果一個鏈接器使用了一個帶有DT_SONAME項名字叫做lib1的共享目標文件以及一個其他路徑名爲/usr/lib/lib2的共享目標文件，那麼可執行文件中將會包含lib1以及/usr/lib/lib2依賴列表。

如果一個共享目標文件具有一個或者多個/，例如/usr/lib/lib2或者directory/file，那麼動態鏈接器會直接使用那個字符串來作爲路徑的名字。如果名字中沒有/，比如lib1，那麼以下的三種機制給出了共享目標文件搜索的順序。

- 首先，動態數組標記DT_RPATH可能會給出一個包含一系列以:分割的目錄的字符串。例如 /home/dir/lib:/home/dir2/lib: 告訴我們先在`/home/dir/lib`目錄搜索，然後再在`/home/dir2/lib`搜索，最後在當前目錄搜索。

- 其次，進程環境變量中的名叫LD_LIBRARY_PATH的變量包含了一系列上述所說格式的目錄，最後可能會有一個;，後面跟着另外一個目錄列表後面跟着另外一個目錄列表。這裏給出一個例子，效果與第一個所說的效果相同

  - LD_LIBRARY_PATH=/home/dir/lib:/home/dir2/lib:
  - LD_LIBRARY_PATH=/home/dir/lib;/home/dir2/lib:
  - LD_LIBRARY_PATH=/home/dir/lib:/home/dir2/lib:;

  所有的LD_LIBRARY_PATH中的目錄只會在搜索完DT_RPATH纔會進行搜索。儘管有一些程序（如鏈接編輯器）在處理;前後的列表的方式不同，但是動態鏈接器處理的方式完全一樣，除此之外，動態鏈接器接受分號表示語法，正如上面所描述的樣子。

- 最後，如果以上兩組目錄無法定位期望的庫，則動態鏈接器搜索`/usr/lib` 路徑下的庫。

注意

> **爲了安全性，對於`set-user` 以及 `set-group` 標識的程序，動態鏈接器忽略搜索環境變量（例如`LD_LIBRARY_PATH`），僅僅搜索`DT_RPATH`指定的目錄和`/usr/lib`。**