# Generate Wordcloud through jieba library

First of all, to generate wordcloud, u need to install some necessary library: ``jieba``, ``wordcloud``. Then, through the QQ message manager, download the message to the local. Modify the file location in the code, the word cloud image will be generated to the selected position.

```python
import jieba
import wordcloud
import imageio

newtext = []
for word in open('***massage adress***', 'r', encoding='utf-8'):
    tmp = word[0:4]
    if (tmp == "2021" or tmp == "===="or tmp == "2022"):
        continue
    tmp = word[0:2]
    if (tmp[0] == '[' or tmp[0] == '/'or tmp[0] == '@'):
        continue
    newtext.append(word)
with open('***text adress***', 'w', encoding='utf-8') as f:
    for i in newtext:
        f.write(i)

remove_words = [u'的', u'，',u'和', u'是', u'随着', u'对于', u'对',u'等',u'能',u'都',u'。',u' ',u'、',u'中',u'在',u'了',u'通常',u'如果',u'我们',u'需要',u'我',u'你',u'？',u"",u" ",u"就",u"不","啊",u"吧",u"也",u"不是",u"就是",u"什么",u"怎么",u"这个",u"这么",u"一个",u"还是",u"可以",u"表情",u"但是",u"还有",u"现在",u"然后",u"没有",u"感觉",u"好像",u"自己",u"知道",u"那个",u"撤回",u"一条",u"消息",u"时候",u"应该",u"直接",u"已经"]

mask = imageio.imread("***mask_image adress***")
f = open("***text adress***", 'r', encoding='utf-8')
t = f.read()
f.close()
ls = jieba.lcut(t)
lss=[]
for word in ls:
		if word not in remove_words:
			if len(word) > 1:
				lss.append(word)
txt = " ".join(lss)
w = wordcloud.WordCloud(font_path = "C://Windows/Fonts/msyh.ttc", mask = mask, width = 1000, height = 700, background_color = "white")
w.generate(txt)
w.to_file("***wordcloud adress***")
```

After generating the word cloud, you will find that the generated image includes a lot of "common words", just add the common words to ``remove_words``.
