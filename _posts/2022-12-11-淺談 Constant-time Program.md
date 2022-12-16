---
title: 淺談 Constant-time Program | 資安小講堂
date: 2022-12-11
categories: [Cybersecurity]
math: true
---

# 前言

今天想跟大家分享一個資安領域的有趣主題：Constant-time Program。目前這個主題還找不到中文相關的翻譯，容我自作主張將它翻譯為「常數時間程式」，但本篇還是主要以英文表示這個詞。這是一種防止計時攻擊的寫程式手法，這裡我會先介紹什麼是計時攻擊，之後再來談談對付它的方法

> 💡 本文不要求任何資安相關知識，只要會讀基本的程式即可理解

# 計時攻擊 Timing Attack

今天考慮一個情景，你想要寫一個需要密碼來啟動的應用程式，於是你寫了這段程式來檢查使用者輸入的密碼是否正確

```c
bool check_password_correct (char input[16], char password[16]){
    for (int i = 0; i < 16; i++){
        if (input[i] != password[i])
            return false;
    }
    return true;
}
```

這是一個很基本的字串比對函數，由前到後比對一個個字元是否正確。若輸入字串有一個字元不 match，則馬上回傳 false、代表輸入的密碼錯誤。反之若所有字元都正確，則回傳 true

這段程式雖然能正確比對輸入字串與密碼字串，但卻有個問題。這個函數會在發現第一個錯誤字元時就馬上返回，如果今天有個攻擊者能夠很精確地測量程式執行的時間，那麼時間就會洩漏出我們密碼的內容

舉個例子，今天當我們的password是「qwer….」時，攻擊者嘗試輸入「aaaa….」、「baaa….」直到「zaaa….」，並記錄下每次程式執行的時間，結果如下

| Input | Execution Time |
| --- | --- |
| aaaa… | 10 ns |
| baaa… | 10 ns |
| … |  |
| qaaa… | **13 ns** |
| … |  |
| zaaa… | 10 ns |

對於所有嘗試的輸入，程式都在 10 ns 內就完成並顯示密碼錯誤，但只有在輸入「qaaa…」時花了 13 ns。如果你回去看一下程式的原始碼就會知道這是因為對於所有輸入、第一個字元就錯了，因此在迴圈的第一次就會直接返回，但當輸入是「qaaa…」時，第一個字元是正確的，因此程式在迴圈的第二次 iteration 才會返回

而只要用時間的資訊就足以讓攻擊者知道真正密碼的第一個字元是「q」，而他接著就可以使用同樣方法去找出之後的字元。密碼有16字元，本來要使用暴力法的話需要猜 $256^{16} ≈ 3.4\times 10^{38}$次才能破解，但使用上述的這種手法、攻擊者只需要嘗試 $256 \times 16 = 4096$ 次即可破解密碼，密碼的安全性大大地下降了

那我們有什麼方法能夠修好這段程式呢？

## 解決方案 1: Single return

第一種最直接的想法是：我們不要提早返回就好。使用一個變數來記錄回傳值，然後等到最後才 return 就好。就像這樣：

```c
bool check_password_correct (char input[16], char password[16]){
    bool answer = true;
    for (int i = 0; i < 16; i++){
        if (input[i] != password[i])
            answer = false;
    }
    return answer;
}
```

但如果你再仔細想一下，這樣做當迴圈中比對的字元不批配時，該次 iteration 會有兩行程式被執行，也就是「if (input[i] != password[i])」和「answer = false」，但當字元相符的話，只有 if 那行會被執行，所以我們依然沒有修復時間會洩漏資訊的這個問題

## 解決方案 2: Random sleep

另一個可能的提案是：不如我們就在程式結束之前讓它暫停一段隨機長度的時間，就像這樣：

```c
bool check_password_correct (char input[16], char password[16]){
    bool answer = true;
    for (int i = 0; i < 16; i++){
        if (input[i] != password[i])
            answer = false;
    }
    sleep(random());
    return answer;
}
```

這個方法雖然會讓程式變慢，但更大的問題是攻擊者其實只要嘗試多次並取執行時間的平均即可讓隨機的效果消失，因此我們依然暴露在危險之中

## 解決方案 3: Sleep until

另一個解法可能是：不如我們不要暫停隨機長度的時間、而是暫停到一個特定的時間，如此一來程式返回的時間點會固定在某個地方。程式碼可能像這樣：

```c
bool check_password_correct (char input[16], char password[16]){
    float expected_exe_time = 3.0;
    float start_time = time();

    bool answer = true;
    for (int i = 0; i < 16; i++){
        if (input[i] != password[i])
            answer = false;
    }

    float end_time = time();
    sleep(expected_exe_time - (end_time - start_time));
    return answer;
}
```

但這個方案的問題是我們必須要提前知道程式大概會執行多久。如果期望執行時間設得太長、程式的效率會變很差，反之設定的時間太短則暫停就完全沒效果。但程式執行的時間事實上是很難在你寫程式的時候預測的，畢竟隨著你編譯器的最佳化、各種底層硬體的加速機制、執行時系統的繁忙程度等等等等，有太多東西會影響你的執行時間了。因此這個方法也不太實際

## 正確的解決方案: Constant-time programming

在掙扎了許多時間後，資安社群的共識下正確的解法可能只有以下這種

```c
bool check_password_correct (char input[16], char password[16]){
    bool answer = true;
    for (int i = 0; i < 16; i++){
        answer = answer && (input[i] == password[i]);
    }
    return answer;
}
```

這裡迴圈裡面的那行程式碼白話文說就是如果 input 的字元和 password 的不批配的話就把 answer 設成 false，意思和我們解決方案1的程式意思一樣，但如此一來每次 iteration 都會執行2個指令、不論該字元有沒有批配

總結來說，這種使用「時間」這個資訊去得到一些程式或硬體的內部資訊這種手法就叫做「計時攻擊 (timing attack)」，而防止這種攻擊的手法稱作「Constant-time Programming」

# Constant-time Programming

當人們在討論 Constant-time programming 時，我們會把程式的輸入參數分為 secret variable 和 public variable。Constant-time programming 講的並不是讓程式總是在固定的時間內結束，而是要讓程式的執行時間獨立於 secret variable。舉例來說，在對稱式加密函式庫中，對於不同內容或長度的密文 (ciphertext)，解密所花的時間可以有變化，但執行時間需要獨立於密鑰 (secret key) ，執行時間才不會洩漏關於密鑰的資訊

另外，雖然這個寫程式的方式並沒有限制應用在哪方面，但在學術界大家探討的對象多半是加密函式庫。因為當加密函式庫受到計時攻擊而被破解，加密的密鑰被洩漏、影響層面會非常大。以大家每天上網都在用的 https 協議為例，若其加密函式庫（可能是 OpenSSL 或 LibreSSL）受攻擊而被破解，我們每天在網路上傳的密碼、資料、信用卡號碼等等全部都會被駭客看光

## Condition to reach constant-time

具體來說要怎麼寫程式才能讓程式成為 constant-time 呢？這邊歸納出4個條件會破壞程式的 constant-time 性質

1. Control flow depending on secret data
    
    在我最初提供的例子裡面，我們有個 if statement 的條件比較了 input 字串和該被視為 secret 的 password 字串，因此下一行被執行的程式碼會取決於秘密變數的內容，也就造成了執行時間的差異
    
    要避免洩漏秘密，我們的 control flow 不能依賴於秘密的內容。更白話來說就是 if、else、while loop、for loop 等等的判斷條件都不能和秘密變數相依，例如這樣的程式就不安全：
    
    ```c
    if (input[i] != password[i])
    	  // do something
    ```
    
2. Accessing memory with address depending on secret data
    
    因為 cache 的存在，存取記憶體的不同位址會花不同時間，也因此要避免存取的位置和秘密變數相依，例如：
    
    ```c
    x = array[secret]
    ```
    
3. Non-constant-time instruction
    
    有些指令本身就會根據參數的不同而有不同的執行時間，例如 x86 CPU 中的整數除法、又或是在大部分 CPU 中的浮點數運算，因此要避免使用這些指令
    
    ```c
    y = secret / 13
    ```
    
4. Recurrence with parameter depending on secret data
    
    遞迴函數執行的次數會和參數的內容有關，因此如果參數是秘密的話，時間也會洩漏資訊，反例就像：
    
    ```c
    recurrence_function(secret)
    ```