# Chirps_monthly_data-download
daily datas are composite 
#### GEE #####
/************************************************************
CHIRPS DAILY Rainfall → Monthly TOTAL (mm/month)
Year: 2015 (2015-01-01 to 2015-12-31)

Output: 12 Monthly rainfall total images (mm/month)
Export: GeoTIFF to Google Drive

You must import/draw ROI as: roi
************************************************************/

// -------------------- SETTINGS --------------------
var start = ee.Date('2016-01-01');
var end   = ee.Date('2017-01-01');  // end is exclusive

var exportScale = 5566;   // Native CHIRPS resolution (~0.05° ≈ 5.5 km)
var folderName  = "CHIRPS_2015_Monthly_TOTAL_mm";

// -------------------- ROI --------------------
Map.centerObject(roi, 7);
Map.addLayer(roi, {color: "red"}, "ROI");

// -------------------- CHIRPS COLLECTION --------------------
var chirps = ee.ImageCollection("UCSB-CHG/CHIRPS/DAILY")
  .filterBounds(roi)
  .filterDate(start, end)
  .select("precipitation")
  .map(function(img){
    return img.clip(roi)
              .copyProperties(img, ["system:time_start"]);
  });

print("Total daily images in 2015:", chirps.size());

// -------------------- MONTHLY TOTAL FUNCTION --------------------
function monthlyTotal(year, month) {
  var mStart = ee.Date.fromYMD(year, month, 1);
  var mEnd   = mStart.advance(1, "month");

  var monthlyCol = chirps.filterDate(mStart, mEnd);

  var total = monthlyCol
    .sum()   // sum of daily rainfall (mm/day) → mm/month
    .rename("Rainfall_mm")
    .set({
      "year": year,
      "month": month,
      "system:time_start": mStart.millis()
    });

  return total;
}

// -------------------- BUILD 12 MONTHS --------------------
var months = ee.List.sequence(1, 12);

var monthlyIC = ee.ImageCollection.fromImages(
  months.map(function(m){
    return monthlyTotal(2016, m);
  })
);

print("Monthly TOTAL Rainfall 2016:", monthlyIC);

// Preview July
var july = ee.Image(monthlyIC.filter(ee.Filter.eq("month", 7)).first());
Map.addLayer(july, {min: 0, max: 800}, "July 2016 Total Rainfall (mm)");

// -------------------- EXPORT LOOP --------------------
var list = monthlyIC.toList(monthlyIC.size());
var n = monthlyIC.size().getInfo();

for (var i = 0; i < n; i++) {
  var img = ee.Image(list.get(i));
  var year  = img.get("year").getInfo();
  var month = img.get("month").getInfo();
  var mm = (month < 10) ? ("0" + month) : ("" + month);

  var name = "CHIRPS_TOTAL_mm_" + year + "_" + mm;

  Export.image.toDrive({
    image: img,
    description: name,
    folder: folderName,
    fileNamePrefix: name,
    region: roi,
    scale: exportScale,
    maxPixels: 1e13
  });
}
