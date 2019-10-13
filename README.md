# Tutorial_App_NetCore_React

## Mô tả về project: 

Project MovieWrapper là 1 thư viện chung dùng vào mục đích lấy dữ liệu từ các nhà cung cấp phim như: CGV, Galaxy, BHD. Sau đó convert kết quả dữ liệu đó theo 1 định dạng mặc định duy nhất và trả kết quả như: danh sách phim đang chiếu, chi tiết 1 bộ phim, địa điểm và thời gian chiếu của phim đó.

## Cấu trúc project và business code:

* Bao gồm project library thực hiện việc lấy dữ liệu từ các nhà cung cấp và convert về chung định dạng. Project test dùng để kiểm tra các business code.

![Annotation 2019-10-13 121347](https://user-images.githubusercontent.com/36435846/66711286-21973f80-edb3-11e9-9399-45bfca1c2d9e.png)

** Project MovieWrapper : Có các model dùng để định dạng dữ liệu, các service dùng để xử lý các chi tiết business theo từng nhà cung cấp, helpers là nơi cung cấp các business chung cho service.

** Project MovieWrapper.Test : bao gồm các unit test cho các service của những nhà cung cấp.

* Để lấy được url để lấy dữ liệu từ các nhà cung cấp thì cần dùng tới "Charles tool". Charles tool được dùng để xem những request cũng như respone khi chúng ta tương tác với trình duyệt hoặc mobile app. Thông tin và cách sử dụng Charles tool: https://medium.com/@hackupstate/using-charles-proxy-to-debug-android-ssl-traffic-e61fc38760f7

* Project library MovieWrapper :
 - Với các interface như IVendorGalaxyService, IVendorCGVService, IVendorBHDService sẽ có các method như nhau nhưng kiểu trả về dữ liệu khác nhau ví dụ:
```
public interface IVendorGalaxyService
    {
        List<MovieGalaxy> GetShowingMovie();
        MovieGalaxy GetDetail(string id);
        SessionMovie GetSessionMovie(string id, DateTime date);
    }
```

- Nhưng trong mỗi VendorGalaxyService, VendorCGVService, VendorBHDService được implements bởi interface tương ứng trên thì có những kiểu xử lý khác nhau:
##### VendorGalaxyService:
* Đây là hàm trả về địa điểm và thời giam chiếu phim: 
```
public SessionMovie GetSessionMovie(string id, DateTime date)
        {
            var result = MovieWrapperHelper.GetAsync($"{_baseUrl}/session/movie/{id}").Result;
            var sessionMovieGalaxys = JsonConvert.DeserializeObject<List<SessionMovieGalaxy>>(result);
            var locations = (from sessionMovie in sessionMovieGalaxys
                             where sessionMovie.Dates != null &&
                                   sessionMovie.Dates.Any(d => d.ShowDate.Equals(date.ToString("dd/MM/yyyy")))
                             select sessionMovie.Name).ToList();

            if (locations.Count == 0) return null;

            return new SessionMovie
            {
                Date = date,
                Locations = locations
            };
        }
```
Dòng code:  ``` var result = MovieWrapperHelper.GetAsync($"{_baseUrl}/session/movie/{id}").Result; ``` truyền url lấy được từ Charles tool vào phương thức GetAsync() từ class MovieWrapperHelper để lấy kết quả trả về là json.

```
var sessionMovieGalaxys = JsonConvert.DeserializeObject<List<SessionMovieGalaxy>>(result);
var locations = (from sessionMovie in sessionMovieGalaxys
                 where sessionMovie.Dates != null &&
                       sessionMovie.Dates.Any(d => d.ShowDate.Equals(date.ToString("dd/MM/yyyy")))
                 select sessionMovie.Name).ToList();

            
```
Chúng ta lấy kết quả sau đó convert json về Object bằng cách sử dụng JsonConvert.DeserializeObject.

```
var locations = (from sessionMovie in sessionMovieGalaxys
                       where sessionMovie.Dates != null &&
                             sessionMovie.Dates.Any(d => d.ShowDate.Equals(date.ToString("dd/MM/yyyy")))
                       select sessionMovie.Name).ToList();
if (locations.Count == 0) return null;

return new SessionMovie
{
    Date = date,
    Locations = locations
};
```
Dùng Linq để lấy ra danh sách tên những nơi chiếu bộ phim đó theo ngày mà chúng ta mong muốn. Sau đó gán giá trị lấy được vào object SessionMovie và trả về kết quả.

##### VendorBHDService: 
Cũng là hàm trả về địa điểm và thời giam chiếu phim nhưng xử lý khác:

```
public SessionMovie GetSessionMovie(string id, DateTime date)
        {
            var resultSession = MovieWrapperHelper.GetAsync($"{_baseUrl}/sessions?filmId={id}&start={date.ToString("yyyy-MM-dd")}").Result;
            var resultCinema = MovieWrapperHelper.GetAsync($"https://booking.bhdstar.vn/WSVistaWebClient/RESTData.svc/cinemas").Result;

            string[] separator = { "\\r" };
            var cinemaXMLs = resultCinema.Split(separator, StringSplitOptions.RemoveEmptyEntries);
            var cinemaBHDs = JsonConvert.DeserializeObject<List<CinemaBHD>>(resultSession);
            var cinemaIds = cinemaBHDs.Select(m => m.CinemaId).Distinct().ToList();
            var locations = new List<string>();

            foreach (var cinemaXML in cinemaXMLs)
            {
                var strList = cinemaXML.Split(new[] { '|' }, StringSplitOptions.RemoveEmptyEntries);
                var strDistinct = strList.Distinct().ToList();
                if (strDistinct.Count != 2 ||
                    !cinemaIds.Any(cinemaId => cinemaId.Equals(strDistinct.FirstOrDefault()))) continue;

                locations.Add(strDistinct.ElementAt(1));
            }

            return new SessionMovie
            {
                Date = date,
                Locations = locations
            };
        }
```
Vì nhà chung cấp BHD chỉ trả id của rạp chiều phim trong 
``` var resultSession = MovieWrapperHelper.GetAsync($"{_baseUrl}/sessions?filmId={id}&start={date.ToString("yyyy-MM-dd")}").Result;```. Nên chúng ta cần gọi thêm 1 đường dẫn khác để lấy danh sách rạp chiếu phim: 
```var resultCinema = MovieWrapperHelper.GetAsync($"https://booking.bhdstar.vn/WSVistaWebClient/RESTData.svc/cinemas").Result; ``` 
nhưng dánh sách rạp chiều phim được trả về là 1 đoạn chuỗi như sau : 
  ```
  "9\r0000000001|0000000001|BHD Star 3.2|BHD Star 3.2|\r0000000002|0000000002|BHD Star Bitexco|BHD Star Bitexco|\r0000000009|0000000009|BHD Star Discovery|BHD Star Discovery|\r0000000008|0000000008|BHD Star Hue||\r0000000006|0000000006|BHD Star Le Van Viet|BHD Star Le Van Viet|\r0000000003|0000000003|BHD Star Pham Hung|BHD Star Pham Hung|\r0000000007|0000000007|BHD Star Pham Ngoc Thach||\r0000000004|0000000004|BHD Star Quang Trung||\r0000000005|0000000005|BHD Star Thao Dien||\r"
```
Vì vậy cần cắt chuỗi đoạn chuôi này để lấy id và name của rạp chiều phim: 
```
string[] separator = { "\\r" };
var cinemaXMLs = resultCinema.Split(separator, StringSplitOptions.RemoveEmptyEntries);
var cinemaBHDs = JsonConvert.DeserializeObject<List<CinemaBHD>>(resultSession);
var cinemaIds = cinemaBHDs.Select(m => m.CinemaId).Distinct().ToList();
var locations = new List<string>();

foreach (var cinemaXML in cinemaXMLs)
{
    var strList = cinemaXML.Split(new[] { '|' }, StringSplitOptions.RemoveEmptyEntries);
    var strDistinct = strList.Distinct().ToList();
    if (strDistinct.Count != 2 ||
        !cinemaIds.Any(cinemaId => cinemaId.Equals(strDistinct.FirstOrDefault()))) continue;

    locations.Add(strDistinct.ElementAt(1));
}
```

* Project MovieWrapper.Test:

Chúng ta sử dụng Moq 






