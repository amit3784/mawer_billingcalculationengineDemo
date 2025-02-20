<template>
<div class="fileuploadbody">
  <div>
    <h3 style="background-color: #6B88CF;">Billing Demo</h3>
  </div>
  <div style="display: flex;background-color:#889218">
    <b style="padding-right: 10px;">Upload File:</b>
    <input type="file" accept=".xlsx,.xls,.csv" @change="handleFileUpladMethod" />
    <b style="padding-right: 10px;padding-left: 10px;"> Select Method : </b>
    <select class="form-control" name="fileuploadmethods" id="fileuploadmethod" v-model="fileuploadtype"
      @change="setType">
      <option value="api">API</option>
      <option value="JS">SheetJS</option>
    </select>
    <div v-if="showType != false">
      <div>
        <b style="padding-right: 10px;padding-left: 10px;"> Choose Chart type: </b>
        <select class="form-control" name="chart-type" id="chart-type" v-model="chartType" @change="setType">
          <option value="bar">Bar</option>
          <option value="line">Line</option>
          <option value="doughnut">Doughnut</option>
          <option value="pie">Pie</option>
          <option value="polarArea">Polar Area</option>
          <option value="radar">Radar</option>
        </select>
      </div>
    </div>
  </div>
  <div style="padding-top: 10px;">
    <canvas class="canvas" ref="chartCanvas"></canvas>
  </div>
</div>
</template>

<script>
import * as XLSX from 'xlsx';
import Chart from 'chart.js/auto';
import alasql from 'alasql';
export default {
  data() {
    return {
      fileuploadtype: "JS",
      chartType: "bar",
      showType: false,
      type: 'bar',
      chartInstance: null,
      data: []
    };
  },
  methods: {
    handleFileUpladMethod(event) {


      if (this.fileuploadtype == 'api') {
        this.handleFileUploadViaAPI(event);
      }
      else {
        this.handleFileUpload(event);
      }
    },
    setType() {
      this.type = this.chartType;

      const chartData = this.chartInstance.config.data;
      //alert(chartData);
      this.createChart(this.data);
      if (this.fileuploadtype == 'api') {
        handleFileUploadViaAPI(event);
      }
      else {
        handleFileUpload(event);
      }

    },
    createChart(chartData) {


      this.showType = true;
      const typeChart = this.type;

      const canvas = this.$refs.chartCanvas;
      const ctx = canvas.getContext('2d');
      // alert(ctx);
      // alert(this.chartInstance);
      if (this.chartInstance != null) {
        this.chartInstance.destroy();
      }

      const labels = Array.from(chartData).map(row => row.clientid + '-' + row.portfolioid + ' ( Fee percentage: ' + row.feepercentage + ')');
      const data = Array.from(chartData).map(row => row.fees);
      const feepercentage = Array.from(chartData).map(row => row.feepercentage);

      this.chartInstance = new Chart(ctx, {
        type: typeChart,
        data: {
          labels,
          datasets: [{
            label: "Fees",
            data,
            backgroundColor: ['red', 'violet', 'indigo', 'blue', 'green', 'yellow', 'orange', 'rgba(75, 192, 192, 1)', 'rgba(154,73,93,.5)', 'rgba(254,73,93,.5)', 'rgba(75, 192, 192, 0.2)'],
            borderColor: ['rgba(75, 192, 192, 1)', 'rgba(154,73,93,.5)', 'rgba(254,73,93,.5)', 'rgba(75, 192, 192, 0.2)'],
            borderWidth: 1
          }]
        }
      });
    },

    async handleFileUploadViaAPI(event) {
      event.preventDefault();
      this.showType = true;
      let formData = new FormData();
      formData.append('file', event.target.files[0]);
      // POST request using fetch()
      fetch("https://localhost:7065/api/FileUpload", {

        // Adding method type
        method: "POST",

        // Adding body or contents to send
        body: formData,


      })

        // Converting to JSON
        .then(response => response.json())

        // Displaying results to console
        .then(json => {


          this.data = json;
          this.createChart(this.data);
          event.target.reset();
        });






    },
    handleFileUpload(event) {
      const file = event.target.files[0];

      const reader = new FileReader();

      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        const workbook = XLSX.read(data, { type: 'array' });

        const clientBillingSheet = workbook.SheetNames[0];
        const portfolioSheet = workbook.SheetNames[1];
        const assetSheet = workbook.SheetNames[2];
        const billingTierSheet = workbook.SheetNames[3];
        const jsonDataClient = XLSX.utils.sheet_to_json(workbook.Sheets[clientBillingSheet], { header: 1 });
        const jsonDataPortfolio = XLSX.utils.sheet_to_json(workbook.Sheets[portfolioSheet], { header: 1 });
        const jsonDataAsset = XLSX.utils.sheet_to_json(workbook.Sheets[assetSheet], { header: 1 });
        const jsonDataBillingTier = XLSX.utils.sheet_to_json(workbook.Sheets[billingTierSheet], { header: 1 });

        // Convert sheet json to Json corrected format.
        const clients = this.JsonConvert(jsonDataClient);

        const portfolios = this.JsonConvert(jsonDataPortfolio);

        const assets = this.JsonConvert(jsonDataAsset);

        const billings = this.JsonConvert(jsonDataBillingTier);


        // compose json output response using akasql.
        

        const result = alasql('SELECT clients.[Client ID] AS ClientId, portfolios.[Portfolio ID] AS PortfolioId,portfolios.[Portfolio Currency] as PortfolioCurrency, \
        assets.[Date] as Date, assets.[Asset Value] AS AssetValue,assets.[Currency] as AssetCurrency,clients.[Billing Tier ID] as TierId \
         FROM ? clients \
         JOIN ? portfolios ON clients.[Client ID] = portfolios.[Client ID] \
         JOIN ? assets ON portfolios.[Portfolio ID]=assets.[Portfolio ID] \
         ORDER BY clients.[Client ID] ', [clients, portfolios, assets, billings]);

        // console.log(result);

        const filteredResult = alasql('SELECT ClientId as clientid,PortfolioId as portfolioid,TierId as tierid, \
        SUM(AssetValue) AS assetvalue FROM ? result \
        GROUP BY ClientId,PortfolioId,TierId', [result])
        //console.log(filteredResult);

        // Apply fees and fee percentage.
        var feesdata = [];

        for (var i = 0; i < filteredResult.length; i++) {
          var feesObject = {};
          let difference = 0;
          let fees = 0;
          let feepercentage = 0;

          var filter = billings.filter(item => item["Tier ID"] === filteredResult[i]['tierid']);
          filter.forEach(element => {

            if (filteredResult[i]['assetvalue'] > element["Portfolio AUM Max ($)"]) {

              difference = parseFloat(element["Portfolio AUM Max ($)"]) - parseFloat(element["Portfolio AUM Min ($)"]);
              fees += (parseFloat(difference)) * (parseFloat(element["Fee Percentage (%)"]));
              //console.log(difference);
              //console.log(parseFloat(element["Fee Percentage (%)"]));
            }
            else {
              //console.log(2);
              difference = parseFloat(filteredResult[i]['assetvalue']) - parseFloat(element["Portfolio AUM Min ($)"]);
              fees += (parseFloat(difference)) * (parseFloat(element["Fee Percentage (%)"]));

              feepercentage = parseFloat(fees * 100 / parseFloat(filteredResult[i]['assetvalue']));
            }

            feesObject.clientid = filteredResult[i]['clientid'];
            feesObject.portfolioid = filteredResult[i]['portfolioid'];
            feesObject.tierid = filteredResult[i]['tierid'];
            feesObject.assetvalue = filteredResult[i]['assetvalue'];
            feesObject.fees = fees;
            feesObject.feepercentage = feepercentage;
          });

          feesdata.push(feesObject);
          //console.log(feesdata);
        };

        this.data = feesdata;
        this.createChart(this.data);
        //event.target.reset();
      };

      reader.readAsArrayBuffer(file);
    },
    JsonConvert(source) {
      var jsonArray = [];
      var headers = source[0];

      for (var i = 1; i < source.length; i++) {
        var rowObject = {};
        for (var j = 0; j < headers.length; j++) {
          rowObject[headers[j]] = source[i][j];
        }
        jsonArray.push(rowObject);
      }
      return jsonArray;
    }
  }
};
</script>

<style scoped>
table {
  width: 100%;
  border-collapse: collapse;
}

th,
td {
  border: 1px solid #ddd;
  padding: 8px;
}

th {
  background-color: #f2f2f2;
}

canvas {
  width: 99%;
  max-height: 700px;
  border: 1px solid rgb(122, 86, 138);
  background-color: #B8D7D2;
}
.fileuploadbody {
   font-family: 'Times New Roman', Times, serif;
   font-weight: normal;
   margin-left: 10px;
}
</style>