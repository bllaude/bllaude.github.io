---
layout: post
title:  PHP array sort key
date:   2020-01-13 22:08:00 +0800
categories: notes
tag: programming
topping: false
---

* content
{:toc}
Encountered a problem: There's an array, where each element contains two sub-elements, time and content, which need to be sorted by the time field.

I remember I already did this thing before, there is a special function, it seems to be `array_multisort` to do it. I checked the manual and found that the example is too obscure and looks very uncomfortable. So simply implement one by yourself, only need to provide a sorting function. The efficiency may be lower, but it is much more comfortable to use. .

```php
// Call example (cmp function can be rewritten by yourself):
mysort($array_name, 0, count($array_name)-1, cmp);

function cmp($i, $j){
    return$i['time'] <$j['time'];
}

functionmysort(&$arr, $s, $e, $func){
    $i = $s; $j = $e; $tmp = $arr[$s];
    if($s <$e){
        while($i != $j){
            while($i <$j && $func($tmp, $arr[$j])) $j--;
            if($i <$j){
                $arr[$i] = $arr[$j];
                $i++;
            }
            while($i <$j && $func($arr[$i], $tmp)) $i++;
            if($i <$j){
                $arr[$j] = $arr[$i];
                $j--;
            }
        }
        $arr[$i] = $tmp;
        mysort($arr, $s, $i- 1, $func);
        mysort($arr, $i+1, $e, $func);
    }
}
```
