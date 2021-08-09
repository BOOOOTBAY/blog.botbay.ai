---
title: Rasa使用——以星座运势为case
author: xiaoshishi
categories: news
tags:
  - home
  - ai
  - dinner
  - event
image: /assets/2021/rasa-usage/rasa.webp
---

> 经过两周的摸索和实验，今天终于把星座运势推进到最后的测试阶段。尽管跑通一个demo会觉得rasa非常简单，但完成一个function，从分离不同的intent，对话框架设计，设置rasa pipeline，具体月份的读取，考虑边界和异常情况，才发现对rasa的了解仅限于冰山一角。

# 借助BotSociety设计story

## 围绕API功能和用户提问角度来写story

**API功能：**

 - 查询总体星座运势
 - 查询爱情指数
 - 查询工作指数
 - 查询财运指数
 - 查询健康指数
 - 查询幸运颜色
 - 查询匹配星座
 - 星座映射到日期
 - 日期映射到星座
 - 安慰等情绪响应 
 
 **用户角度：**
 
 - 知道星座，明确提问
 - 不知道星座，模糊提问，日期映射
 - 查看全部运势
 - 查看具体运势

**设计story：**
![botsociety
](https://img-blog.csdnimg.cn/e6981021bd2b41de80c598d9e531cfc0.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)

# rasa部分

## 中文分词问题

**问题描述：**
![](https://img-blog.csdnimg.cn/83b95a7c81ee42c6b674e3afdeceb50b.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)在rasa train阶段就出现了boundary的报错
![](https://img-blog.csdnimg.cn/cf43d758af1f424c82a119f705910432.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)
理想实体是 `摩羯座` 但是提取出来的是 `摩羯`

**解决方案：**

![请添加图片描述](https://img-blog.csdnimg.cn/7b8727629da64042a31a35752b39b016.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)

[https://github.com/howl-anderson/rasa_chinese](https://github.com/howl-anderson/rasa_chinese)

## 清空slot的memory

**问题描述：**

![请添加图片描述](https://img-blog.csdnimg.cn/516b4d95bd9f41fc8f3dea98f8e01eca.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)

上图中，bot先询问了处女座的爱情指数，使starsign_type的槽位被填充入处女座，但是在新的一轮对话中，槽位没有被清空，影响了接下来的判断。

**解决方案：**
首先笔者在action中将槽位设为None，如下所示

```python
class ResetAction(Action):
    def name(self) -> Text:
        return "action_reset"

    def run(self, dispatcher: CollectingDispatcher,
            tracker: Tracker,
            domain: Dict[Text, Any]) -> List[Dict[Text, Any]]:

        return [SlotSet('starsign_type', None)]
```

然而，并没有起到清空的效果，而是采用 `AllSlotsReset()` 才能达到预期效果，使对话重新开始。经过实验发现 `slotset` 可以设当前slot为某一个值，但是设为None没反应。

```python
# 正确方案：
        return [AllSlotsReset(), Restarted(), SlotSet('starsign_type', None)]
```

## 日期星座映射

**问题描述：**
由于一些用户不知道自己的星座，所以有必要在星座运势查询中设置星座日期映射的环节。用户的查询方式有多种，例如"7月2" "7 2" "七月二号" “7/2”
**解决方案一：否定** 
用entities/slot（type = list）的方式将用户输入的信息中所有的数字提取出来，在action中进行星座匹配，再将结果返回给用户。
但是在实际操作中，无法将所有的数字提取出来，有些时候只能取到一个数字，非常不稳定，且中文数字提取不出来。
**解决方案二：**
猜想可能是entities的部分没有很好的提取到， `月` 和 `日` 虽然会共享一段数字（1-12，一-十二），但仍需要更明显的区分，于是用rasa中的role来区分。

```yaml
我出生于[7]{"entity": "date", "role": "month"}月[2]{"entity": "date", "role": "day"} .
```

不过，也没有什么用。叹气。

 **解决方案三：**
 我愿称之为，**没有python扶不起的墙**。
 思路是，把用户这句话读取，在action这个自由度很高的部分，利用python的正则来处理。题外话: 虽然rasa也有正则，但是总会出些bug，中文不适配等问题，所以选择熟练度和容易指数高的python好了。

```python
class ReturnStarAction(Action):
    def name(self) -> Text:
        return "action_return_star"

    def checkdate(self, tracker):
        inp = tracker.latest_message['text']
        print(inp)
        number = re.compile(r'([一二三四五六七八九零十]+|[0-9]+)')
        pattern = re.compile(number)
        all = pattern.findall(inp)
        print(all)
        date_status = False
        date_accurate = False
        if len(all) != 2:
            return date_status, None, None, None
        else:
            date_status = True
            month = all[0]
            day = all[1]
            if not month.isdigit():
                month = pycnnum.cn2num(month)
            if int(month) > 0 and int(month) < 13:
                date_accurate = True
            if not day.isdigit():
                day = pycnnum.cn2num(day)
            if int(day) > 0 and int(day) < 32:
                date_accurate = True
            return date_status, date_accurate, month, day

    def findstar(self, month, day):
        daylist = [20, 19, 21, 20, 21, 22, 23, 23, 23, 24, 23, 22]
        starlist = ['摩羯座', '水瓶座', '双鱼座', '白羊座', '金牛座', '双子座',
                    '巨蟹座', '狮子座', '处女座', '天秤座', '天蝎座', '射手座']
        if int(day) < daylist[int(month) - 1]:
            return starlist[int(month) - 1]
        else:
            return starlist[int(month)]

    def run(self, dispatcher: CollectingDispatcher,
            tracker: Tracker,
            domain: Dict[Text, Any]) -> List[Dict[Text, Any]]:
        date_status, date_accurate, month, day = self.checkdate(tracker)
        if not date_status or not date_accurate:
            dispatcher.utter_message(text=f'{"我寻思着这日期只应天上有，要不您说个靠谱的？"}')
            return []
        else:
            star = self.findstar(month, day)
            dispatcher.utter_message(text=f"原来你是{star}~！")
            return [SlotSet('starsign_type', star)]

```

**解决方案四：**
我愿称之为，**逃避策略**
当识别到intent是用户在询问星座日期，就可以直接返回一张图，让用户自行查看即可。

## 关键点设为entities

一开始我并没有把恋爱指数，工作指数，幸运色等作为关键词，而是普通的语料信息。修改后能够更好的提取到关键信息。

```yaml

### 修改前

- intent: ask_horoscope
  examples: |
    - [白羊座](starsign_type)的爱情运势
    - [金牛座](starsign_type)的爱情
    - 我的桃花运咋样
    - 我的恋爱指数

### 修改后

- intent: ask_horoscope
  examples: |
    - [白羊座](starsign_type)的[爱情运势](love)
    - [金牛座](starsign_type)的[爱情](love)
    - 我的[桃花运](love)咋样
    - 我的[恋爱指数](love)
```

## 合并intent

合并相似度高的intent，intent变少了，命中正确率也相应提高。反之，细分如果太多，模型不够那么好，并不会达到预期效果，容易出现intent识别错位的情况。

## 合并path，用slot分流

因为合并了intent，那就意味着出现了公共path，path的一头是intent，一头是action，而决定走向哪一个action，就用slot槽位的填补来确定。这里遵循的原则是尽量走公共path，这也是出于对模型的适应，提高命中率，且用槽位判断可以更精准一些。

![请添加图片描述](https://img-blog.csdnimg.cn/99c0f19a46ed45cdb42df5fba369a759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)

## Rasa测试

按照官方文档[https://rasa.com/docs/rasa/testing-your-assistant](https://rasa.com/docs/rasa/testing-your-assistant)进行测试。
将nlu按照4:1切分成训练集和测试集进行交叉验证。

 - 查看Data and Stories是否有效
 - 切分nlu
 - test nlu
 - 查看intent/slot混淆矩阵
 

![请添加图片描述](https://img-blog.csdnimg.cn/8e6a240f91264052acd17af8ace112b0.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)

![请添加图片描述](https://img-blog.csdnimg.cn/cd2e4701b8f441abb01506f75cb85401.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQ0NTE0Mg==,size_16,color_FFFFFF,t_70)

由于这样的测试数据集都来自写好的nlu，所以还是需要在交互页面中人工测试。
