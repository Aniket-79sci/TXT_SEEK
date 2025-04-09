<script>
	import Button from '$lib/components/ui/button/button.svelte';
	import Progress from '$lib/components/ui/progress/progress.svelte';
	import {onMount} from 'svelte'


	import { createWorker } from 'tesseract.js';

	import { GlobalWorkerOptions, getDocument } from "pdfjs-dist";
    // import "pdfjs-dist/web/pdf_viewer.css";
    // import * as pdfjs from "pdfjs-dist/web/pdf_viewer.mjs";

	import PDFMerger from 'pdf-merger-js/browser';

	import { Input } from "$lib/components/ui/input/index.js";
  	import { Label } from "$lib/components/ui/label/index.js";

    let workerSrc = "https://unpkg.com/pdfjs-dist@5.0.375/build/pdf.worker.min.mjs";



	let file_img;


	let pdf;

	let pdf_file_url;

	let images;

	let files_remaining_percentage = 0;



	// used in complete(), used to recognize text
	async function recognizeText(images,index,array_length) {
    
		console.log(images);
		try {
			const worker = await createWorker('eng', 1, {
				// workerPath:workerFile,
				// // langPath:langData,
				// corePath:coreData,
				logger: (m) => {
					console.log(m);
					if(m.status = ""){}
				}
				// gzip : false,
				// workerBlobURL:false
			});
			const { data } = await worker.recognize(
				images,
				{ pdfTitle: `part_${index}` },
				{ pdf: true }
			);
			console.log(data.text);

			pdf = data.pdf;

			worker.terminate();

			files_remaining_percentage += (1/array_length) * 100; 
			return Promise.resolve({
										"pdf_raw":data.pdf,
										"index":index
									});
		} catch (err) {
			console.error(err);
			return Promise.reject(err);
		}
	}

	// function downloadPdf() {
	// 	const blob = new Blob([new Uint8Array(pdf)], { type: 'application/pdf' });

	// 	const link = document.createElement('a');
	// 	if (link.download !== undefined) {
	// 		const url = URL.createObjectURL(blob);
	// 		link.setAttribute('href', url);
	// 		link.setAttribute('download', file_img[0].name.split('.')[0]);
	// 		link.style.visibility = 'hidden';
	// 		document.body.appendChild(link);
	// 		// link.click();
	// 		document.body.removeChild(link);
	// 		pdf_file_url = url;
	// 	}
	// }

	//used in complete(), used to convert each images to pdfs with text included
	async function convertPdfToImages(file) {

		GlobalWorkerOptions.workerSrc = workerSrc;

		const reader = new FileReader();
		reader.readAsArrayBuffer(file);

		return new Promise((resolve) => {
			reader.onload = async () => {
				const arrayBuffer = reader.result;
				const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
				let pages = [];

				for (let i = 1; i <= pdf.numPages; i++) {
					const page = await pdf.getPage(i);
					const viewport = page.getViewport({ scale: 2 });
					const canvas = document.createElement('canvas');
					const context = canvas.getContext('2d');
					canvas.width = viewport.width;
					canvas.height = viewport.height;

					await page.render({ canvasContext: context, viewport }).promise;
					pages.push(canvas.toDataURL('image/png'));
				}

				resolve(pages);
			};
		});
	}

	// used in merge_pdfs(), used to convert the uint8array data to blobs(files) to merge them;
	function raw_data_to_files(raw_data,fileName) {
		const blob = new Blob([new Uint8Array(raw_data)], { type: 'application/pdf' });

		blob.name = fileName;
		blob.lastModifiedData = new Date();
		
		return blob;
	}
	
	// used in complete(), used to merge separate pdfs in a single pdf.
	async function  merge_pdfs(pdfs_detailed_array) {
		const merger = new PDFMerger();

		pdfs_detailed_array.map(el=>{
			el.pdf_file = raw_data_to_files(el.pdf,`${el.index}.pdf`);
		})

		// pdfs_detailed_array.forEach(async (el)=>{
		// 	await merger.add(el.pdf_file);
		// })
		
		for(let i = 0;i<pdfs_detailed_array.length;i++){
			await merger.add(pdfs_detailed_array[i].pdf_raw);

			
		}

		await merger.setMetadata({
			title : `${file_img[0].name.split('.')[0]}`
		})

		let merged_pdf = await merger.saveAsBlob()
console.log("error not above");
		

		const link = document.createElement('a');
		if (link.download !== undefined) {

			const url = URL.createObjectURL(merged_pdf);

			// ___ the following is for downloading the file___
			// link.setAttribute('href', url);
			// link.setAttribute('download', file_img[0].name.split('.')[0]);
			// link.style.visibility = 'hidden';
			// document.body.appendChild(link);
			// link.click();
			// document.body.removeChild(link);


			pdf_file_url = url;// this shows the pdf file on iframe or can be used in pdfjs
		}


	}

	// initiator function calls most of the functions.
	async function complete() {
	file_img = document.querySelector("#picture").files
	console.log(file_img);
    files_remaining_percentage = 0;
    files_remaining_percentage += 3;
		let images = await convertPdfToImages(file_img[0]);
		console.log(images);

		let ocr_pdfs = [];

		for(let i = 0;i<images.length;i++){
			await recognizeText(images[i],i,images.length).then((data_json) => {
				// downloadPdf();
				ocr_pdfs.push(data_json);

			});
		}

		ocr_pdfs.sort((a,b)=>{
			return a.index - b.index;
		})

		
		merge_pdfs(ocr_pdfs);
		
		console.log(ocr_pdfs);
		// console.log(ocr_pdfs);


	}

	// on:input={() => {
	// 		recognizeText(file_img[0]);
	// 	}}

	let viewerUrl ;	
	$:{
		if(!!pdf_file_url){
			viewerUrl = `/pdfjs/web/viewer.html?file=${encodeURIComponent(pdf_file_url)}`;
		}
	}
</script>



<div>
	<div class="flex flex-col items-center justify-center min-h-screen gap-4">

		<!-- Upload section -->
		<div class="flex flex-col md:flex-row gap-3 items-end">
			<div class="grid w-full max-w-sm items-center gap-1.5">
				<Label for="picture" class="text-3xl font-semibold text-center w-full mt-3 mb-3">Upload PDF Below</Label>
				<Input id="picture" type="file" />
			</div>
			<Button on:click={complete}>Load</Button>
		<!-- <Button class="btn" on:click={recognizeText}>recognize</Button>
		<Button class="btn" on:click={downloadPdf}>Download PDF</Button> -->
		</div>

		<div class="py-4 pr-5 w-[90%]">
			<Progress value={files_remaining_percentage} max={100} class=""></Progress>
		</div>

	</div>

	<iframe src={viewerUrl} frameborder="0" title="pdf" class="h-screen w-screen {!!pdf_file_url?"":"hidden"}"></iframe>
</div>