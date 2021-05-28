### Clustering web pages based on structure and style similarity
基于结构和样式相似度 集群 Web 页面

相似性：
- 语义相似性
  - 页面的含义（关键字，主题）
- 语法相似性
  - 页面结构，CSS 样式

相似性检测：
- HTML
- CSS
- (NO) Javascript

1. 结构相似性 -> DOM Tree -> tree edit distance
2. 样式相似性 -> CSS Class names -> Jaccard Similarity

k.structural + (1-k).style -> 相似度矩阵

Shared Near Neighbor alg -> Clusters 


TED 花费巨大

### html-similarity
structural -> sequence comparison of html tags
Style -> CSS Class Name

### HTMLSimilarity
根据结构确认页面相似性

HTML -(Parser)-> DOM Tree -(Hash)-> feature vector

---
### 参考资料

- https://www.slideshare.net/thammegowda/ieee-iri-16-clustering-web-pages-based-on-structure-and-style-similarity?qid=7deea5f8-157d-4e57-a413-16ec7c6a22d9&v=&b=&from_search=1

- https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7785739

- http://oss.wanfangdata.com.cn/www/网页结构相似性确定方法及装置.ashx?isread=true&type=patent&resourceId=CN200910235278.6&transaction=%7B%22id%22%3Anull%2C%22transferOutAccountsStatus%22%3Anull%2C%22transaction%22%3A%7B%22id%22%3A%221373906542646026240%22%2C%22status%22%3A1%2C%22createDateTime%22%3Anull%2C%22payDateTime%22%3A1616399820373%2C%22authToken%22%3A%22TGT-3293954-eKeLAAHc2H35U1DhTm4ljQU9IuvTgexyFFPcwbURw7Pgybn32X-my.wanfangdata.com.cn%22%2C%22user%22%3A%7B%22accountType%22%3A%22Group%22%2C%22key%22%3A%22shjtdxip%22%7D%2C%22transferIn%22%3A%7B%22accountType%22%3A%22Income%22%2C%22key%22%3A%22PatentFulltext%22%7D%2C%22transferOut%22%3A%7B%22GTimeLimit.shjtdxip%22%3A3.0%7D%2C%22turnover%22%3A3.0%2C%22orderTurnover%22%3A3.0%2C%22productDetail%22%3A%22patent_CN200910235278.6%22%2C%22productTitle%22%3Anull%2C%22userIP%22%3A%22116.246.0.93%22%2C%22organName%22%3Anull%2C%22memo%22%3Anull%2C%22orderUser%22%3A%22shjtdxip%22%2C%22orderChannel%22%3A%22pc%22%2C%22payTag%22%3A%22Shibboleth%22%2C%22webTransactionRequest%22%3Anull%2C%22signature%22%3A%22gnE%2FIIly0noi9G352ZyjwzWmvUujurxw0IXFSYgGTgnl1Lp61TfWZk2n8ldSgzPpKU%2BIdccpqep2%5Cnnb1lnkGQQg8rfFObS0wYIO27EvPjD3bBzSB9tTAXBvWpQZkuxqE5M1LkF20HjycHXT3MwYOClvRE%5CnNAhDSRbOj1kapuj7l9g%3D%22%2C%22delete%22%3Afalse%7D%2C%22isCache%22%3Afalse%7D

- https://github.com/SPuerBRead/HTMLSimilarity

- https://github.com/matiskay/html-similarity

- https://github.com/TeamHG-Memex/page-compare