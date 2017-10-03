# Từ hàng tỉ phép so sánh đến 10 giây.
Mình mới làm một dự án nho nhỏ về xử lý dữ liệu cho khách hàng `X`. Dữ liệu không lớn lắm, chỉ vài chục MB nhưng cũng có khá nhiều điều để nói.

Mình viết bài này để chia sẻ lại với anh em cách mà mình đã làm nhé.

# Bài toán
- Trong DB (`MY_DOMAIN`) mình có khoảng 500K domains có dạng `/^[\w]+(\.com)?\.vn$/`
- Hàng ngày mình phải tải dữ liệu từ trên một số trang web nước ngoài về, Khoảng (~4 triệu domains) được lưu trong các files (~20 files khác nhau). Các domains này có dạng bất kỳ, đúng chuẩn tên miền. :)
- Nhiệm vụ
  - 1. Tải 20 files về (~350MB mỗi ngày)
  - 2. Duyệt ~4 triệu domains trong 20 files đấy
  - 3. Kiểm tra xem domain thứ i có nằm trong cái `MY_DOMAIN` không, nếu có thì đưa vào một cái bảng mới `DB_RESULT`
  - 4. Báo kết quả ? domains, ? inserted, ? updated, ? matched

# Tiếp cận bài toán
- Đại ca khách hàng nói rằng 4 triệu đấy nhân với lại 500 ngàn tức là khoảng `2.000.000.000.000` phép so sánh.
- OK. Em cứ tiếp nhận vậy đã. 1 giây xử lý được khoảng 1 triệu phép tính. Tức là con số trên bỏ đi `6` số `0`, mình còn lại `2.000.000` giây. Chết mợ rồi. Nếu 2 triệu giây có nghĩa là: `2 triệu / 60 = 33 ngàn phút`.
- Có nghĩa là rơi vào khoảng `555 giờ` hay chia cho `24` là ra khoảng hơn `23 ngày`
- LOL, `23 ngày` nếu chạy trâu bò => Không ổn tí nào.

# Giải quyết vấn đề
- Suy nghĩ thêm thì sẽ thấy bảng chữ cái và số nó bắt đầu từ `0-9 & a-z`, cho nên. Mình quyết định làm thế này.
  - Chia cái DB mà có `500 ngàn records` của mình ra làm 36 bảng. Mỗi bảng có `n` dòng. Đẹp đẹp là `500.000/36=13,889 records`
  - Mỗi `1 domains` trong `~ 4 triệu domains`, mình sẽ lấy chữ cái đầu tiên và chỉ search trong cái bảng `prefix_${first_char_of_domain}`
  - Ồ, tốc độ có vẻ cải thiện. `4,000,000 * 11,000 = 44,000,000,000` => `2 ngàn tỉ` xuống còn `44 tỉ` => Còn khoảng `0,5 ngày` ~ `12 hours`.
  - `23 ngày` xuống còn `12 giờ` cũng đẹp phết nhưng mà ai chấp nhận cái nồi đấy.
- Tiếp tục suy nghĩ. Xong cmnr, mỗi lần lại phải search xem trong db có cái `domain` ấy không thì không ổn. Vậy làm thế nào bây giờ.
- OK. Mang 36 cái bảng ấy lên 36 cái mảng để search local cho nó nhanh. Lúc này chịu khó tốn ram 1 tí. Nhưng mà chấp nhận được.
- Suy nghĩ, Duyệt `4,000,000` lần qua `11,000` bực lắm. Có cách nào so sánh nhanh hơn không.
- Mình nghĩ đến một cách là. Lúc insert domains xuống database `MY_DOMAIN` ấy. mình quyết định lưu thêm 1 record là `domainWord`. VD: `ltv.vn` thì sẽ có `2 fields`: `domain, domainWord`, records tương ứng là: `ltv.vn, ltv`
- Lúc lấy lên DB mình sẽ có một cái `HashMap` như sau:
```
const domainMap = {
  'ltv': ['.vn'],
  'lucduong', ['.com.vn', '.vn'],
  //... -> Max của cái hash này là 500 ngàn keys
}
```
- Giờ duyệt mỗi `4,000,000` domains
  - VD: `domains[i]` là `ltv.net`
  - Mình chỉ lấy `ltv` và search trong cái `domainMap` của mình.
  - `insertToResultDB({domain: 'ltv', suffixes: domainMap['ltv']})` Lúc này `suffixes` là cái `[.vn]`
- Độ phức tạp giải thuật của `HashMap` khi lookup key là `~O(1)` cho nên tốc độ lookup key sẽ rất nhanh.
- Tuy nhiên mỗi lần thấy trong `HashMap` thì lại insert thì `performance` sẽ giảm lắm. Vì cứ phải connect, insert, next. Mình không hài lòng nên đã chơi cái trò là
- `addDomainToBulk({domain: 'ltv', suffixes: domainMap['ltv']})`
- Kết thúc vòng lặp `4,000,000`, mình quấy `bulkInsert` cái mảng.
- Final result: `~30 seconds` thỉnh thoảng dữ liệu nhiều hơn thì rơi vào khoảng `1 phút`
- Vẫn chưa hài lòng lắm mình quyết định chia cái `domainMap` thành `"36 cái maps"`
```
const domainMaps = {
  //...,
  'l': {
    'ltv': ['.vn'],
    'lucduong': ['.com.vn','.vn']
  },
  'v': {
    'viblo': ['.com.vn', .vn]
  },
  //...,
}
```
- Mục đích là để giảm số lần lookup key của cái map.
- Thực sự là giảm rất nhiều. (cwl)
- Sau đó final result của mình chỉ giao động trong khoảng `10 giây ~ 20 giây`

# Code mẫu
## Hàm thêm dữ liệu vào bảng `MY_DOMAIN`
```
const importDomainsFromExcel = async (domains) => {
  const promises = []
  let tablePrefix = 'MY_DOMAIN_'
  const tbMap = {}
  _.forEach(domains, d => {
      // EX: d = ltv.vn
      let firstChar = d[0]
      let tableNm = `${tablePrefix}${firstChar}`
      let domainWord = d.replace(/(\.com)?\.vn/g, '') // => domainWord = 'ltv'
      if (!!tbMap[tableNm]) {
        tbMap[tableNm].push({
            domain: d,
            domainWord,
          })
      } else {
        tbMap[tableNm] = [{
            domain: d,
            domainWord,
          }]
      }
  })

  Object.keys(tbMap).forEach(tableNm => {
    promises.push(insertDomainsToMyDomain(tableNm, tbMap[tableNm])) // bulkInsert
  })

  return Promise.all(promises)
}
```

## Hàm thêm nhiều domains vào bảng `MY_DOMAIN_X`
```
const insertDomainsToMyDomain = async (tableNm, domains) => {
  // get connection assign to db
  const collection = db.collection(tableNm)
  const bulk = initBulkInsert()
  _.forEach(domains, d => {
      bulk.insert(d)
  })
  return bulk.execute()
}
```

## Hàm đọc dữ liệu từ nhiều bảng `MY_DOMAIN_` và assign vào `HashMap`
```
// get CHARS from constants ['0', '...', '9', 'a', ..., 'z']
const fetchDomainsToHashMap = async () => {
  const domainMap = {}
  let tablePrefix = 'MY_DOMAIN_'
  const promises = []
  _.forEach(CHARS, c => {
    let tableNm = `${tablePrefix}${c}`
    promises.push(fetchDomainsFromCollection(db.collection(tableNm)))
  })

  const domains = Promise.all(promises).then(async (results) => {
      const resDomains = []
      _.forEach(results, _domains => {
          resDomains.concat(_domains)
      })
      return resDomains
  })

  _.forEach(domains, d => {
      let firstChar = d[0]
      if (!!domainMap[firstChar]) {
        if (!!domainMap[firstChar][d.domainWord]) {
          domainMap[firstChar][d.domainWord].push(d.suffixes)
        } else {
          domainMap[firstChar][d.domainWord] = [d.suffixes]
        }
      } else {
        domainMap[firstChar] = {
          [d.domainWord]: [d.suffixes]
        }
      }
  })

  return domainMap
}
```

## Hàm so sánh và insert vào DB
```
const compareAndImport = async (domains4MillionsFromFiles, domainMap) => {
  const bulkPrepare = initBulkInsert()
  _.forEach(domains4MillionsFromFiles, d => {
      let domainWord = d.replace(/(\.com)?\.vn/g, '')
      let suffixes = domainMap[d[0][domainWord]]
      if (!!suffixes) {
        bulkPrepare.insert({
            domain: d,
            suffixes: suffixes
          })
      }
  })

  return bulkPrepare.execute()
}
```

# Ngôn ngữ sử dụng
- `NodeJs v8` -> Code web chính
- `Go Lang v1.8x` -> Một vài hàm import excel, download file, ...


## Kết quả
```
10s ~ 20s
```

## Mẫu DB
`MY_DOMAIN_L`

| domain          | domainWord     |
| :-------------  | :------------- |
| lucduong.com.vn | lucduong       |
| ltv.vn          | ltv            |

`DB_RESULT`

| domain         | suffixes       |
| :------------- | :------------- |
| ltv            | [.ltv]         |

## Kết luận

Một bài viết mang tính chất giải trí. :)
