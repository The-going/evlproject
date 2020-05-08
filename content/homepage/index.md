---
headless: true
title: homepage
---

<div class="home-grid-container">

  <div class="signature-item">
       <a href="#">{{< homeimg src="images/signature.png" alt="" >}}</a>
  </div>

  <div class="overview-item">
      {{% homebutton href="/overview/" icon="fas fa-play" %}}Get started{{% /homebutton %}}
  </div>
 
  <div class="repo-table-item">
      <style>
      #coderef {
	       width: 600px;
      }
      #coderef td {
      	       border-style: none;
	       padding: 2px;
  	       color: #fff;
  	       text-align: center;
      }
      #coderef th {
	       padding: 8px;
  	       text-align: center;
    	       background-color: #fff;
  	       color: #0084ba;
      }
      a:link {
      	 color: #fff;
      }
      a:active {
      	 color: #fff;
      }
      a:visited {
      	 color: #fff;
      }
      a:hover {
      	 color: #00d2ed;
      }
      </style>
      <table id="coderef">
	<col width="40">
      	<col width="25%">
      	<col width="35%">
      <tr>
        <th>Repository</th>
        <th>Branch</th>
        <th>Latest</th>
      </tr>
      <tr>
        <td><a href="https://git.evlproject.org/linux-evl.git">linux-evl (mainline)</a></td>
        <td><a href="https://git.evlproject.org/linux-evl.git/log/?h=evl/master">evl/master</a></td>
        <td><a href="https://git.evlproject.org/linux-evl.git/log/?h=evl/master">{{< param evlTrackingKernel >}}</a></td>
      </tr>
      <tr>
        <td><a href="https://git.evlproject.org/linux-evl.git">linux-evl (LTS)</a></td>
        <td><a href="https://git.evlproject.org/linux-evl.git/log/?h=evl/v5.4">{{< param evlLTSBranch >}}</a></td>
        <td><a href="https://git.evlproject.org/linux-evl.git/tag/?h={{< param evlLTSLatest >}}">{{< param evlLTSLatest >}}</a></td>
      </tr>
      <tr>
        <td><a href="https://git.evlproject.org/libevl.git">libevl</a></td>
        <td><a href="https://git.evlproject.org/libevl.git/log/">master</a></td>
        <td><a href="https://git.evlproject.org/libevl.git/tag/?h={{< param evlLibLatest >}}">{{< param evlLibLatest >}}</a></td>
      </tr>
      </table>
  </div>

</div>
