<script>
  import { Button, ButtonGroup } from "flowbite-svelte";
  import { Spinner, Alert } from "flowbite-svelte";
  import { Heading, P, A } from "flowbite-svelte";
  import { Select, Input, Label, Helper } from "flowbite-svelte";

  import { createDbWorker } from "sql.js-httpvfs";
  import { Sheet } from "lucide-svelte";

  import pTime from "p-time";
  import saveAs from "file-saver";
  // @ts-ignore
  import PapaParse from "papaparse";

  import { LayerCake, Svg } from "layercake";

  // @ts-ignore
  import { create, all } from "mathjs";
  const config = {};
  const math = create(all, config);

  import Line from "./lib/_components/Line.svelte";
  import Area from "./lib/_components/Area.svelte";
  import AxisX from "./lib/_components/AxisX.svelte";
  import AxisY from "./lib/_components/AxisY.svelte";

  import { CodeJar } from "@novacbn/svelte-codejar";

  // @ts-ignore
  import RandExp from "randexp";

  import { writable } from "svelte/store";

  let pageSize = "1024";
  let pageSizes = [
    { value: "512", name: "512" },
    { value: "1024", name: "1024" },
    { value: "2048", name: "2048" },
    { value: "4096", name: "4096" },
    { value: "8192", name: "8192" },
    { value: "16384", name: "16384" },
    { value: "32768", name: "32768" },
  ];

  let testcount = 100;

  let interval = 100;

  let radonRegex = `tt00[1-9]{5}`;

  $: randomValue = new RandExp(radonRegex).gen();

  let sqlQueryTemplate = `SELECT * from titles WHERE title_id = "{{var}}";`;

  $: sampleSqlQuery = sqlQueryTemplate.replace("{{var}}", randomValue);
  let dbUrl =
    "https://nishad.github.io/sql.js-httpvfs-playground/db/imdb-titles-100000_1024_indexed.db";
  import Prism from "prismjs";
  import "prismjs/components/prism-sql";

  const highlight = (code, syntax) =>
    Prism.highlight(code, Prism.languages[syntax], syntax);

  const workerUrl = new URL(
    "sql.js-httpvfs/dist/sqlite.worker.js",
    import.meta.url
  );
  const wasmUrl = new URL("sql.js-httpvfs/dist/sql-wasm.wasm", import.meta.url);

  async function queryDb(sqlQuery, worker) {
    const result = await worker.db.query(sqlQuery);
    const bytesRead = await worker.worker.bytesRead;
    const stats = await worker.worker.getStats();
    return { result, bytesRead, stats };
  }

  let result;
  let querying = false;
  let error = false;
  let errorMessage = "";
  let testing = false;
  let validResult = false;

  const testResult = writable([]);
  const timeTaken = writable([]);
  const bytesRead = writable([]);

  // repeat the query testcount times
  async function runQueryTest() {
    $testResult = [];
    $timeTaken = [];
    $bytesRead = [];
    result = null;
    querying = true;
    error = false;
    testing = true;
    validResult = false;
    let lastSuccesfulBytesRead = 0;
    let lastSuccesfulRequestCount = 0;

    const worker = await createDbWorker(
      [
        {
          from: "inline",
          config: {
            serverMode: "full",
            url: dbUrl,
            requestChunkSize: Number(pageSize),
          },
        },
      ],
      workerUrl.toString(),
      wasmUrl.toString()
    );

    for (let i = 0; i < testcount; i++) {
      // sleep for interval milliseconds
      await new Promise((resolve) => setTimeout(resolve, interval));

      // generate a random value
      const randomValue = new RandExp(radonRegex).gen();

      // replace the variable in the query
      const sqlQuery = sqlQueryTemplate.replace("{{var}}", randomValue);

      let queryData = pTime(queryDb)(sqlQuery, worker);
      await queryData
        .then((data) => {
          const actualBytesRead = math.subtract(
            data.bytesRead,
            lastSuccesfulBytesRead
          );
          lastSuccesfulBytesRead = data.bytesRead;
          const actualRequests = math.subtract(
            data.stats.totalRequests,
            lastSuccesfulRequestCount
          );
          lastSuccesfulRequestCount = data.stats.totalRequests;
          let result = {};
          result.id = i + 1;
          result.timeTaken = queryData.time;
          result.bytesRead = actualBytesRead;
          result.totalRequests = actualRequests;
          result.totalBytes = data.stats.totalBytes;
          result.randomValue = randomValue;
          $testResult = [...$testResult, result];
          $timeTaken = [...$timeTaken, result.timeTaken];
          $bytesRead = [...$bytesRead, actualBytesRead];
          error = false;
          validResult = true;
        })
        .catch((queryError) => {
          error = true;
          errorMessage = queryError.message;
          console.log("Query Error: ", queryError.message);
          console.log(queryError);
        });
    }
    querying = false;
  }
</script>

<main class="pt-8 pb-12 lg:pt-8 lg:pb-12 bg-white">
  <div class=" px-4 mx-auto max-w-screen-xl">
    <div class="p-8">
      <div class="p-6">
        <Heading tag="h2" customeSize="text-4xl font-extrabold"
          >sql.js-httpvfs analyzer</Heading
        >
        <P class="my-4 text-gray-500">
          <code>sql.js-httpvfs</code> is a fork of and wrapper around sql.js to provide
          a read-only HTTP-Range-request based virtual file system for SQLite.</P
        >
        <A href="https://github.com/phiresky/sql.js-httpvfs"
          >Read more
          <svg
            class="ml-1 w-6 h-6"
            fill="currentColor"
            viewBox="0 0 20 20"
            xmlns="http://www.w3.org/2000/svg"
            ><path
              fill-rule="evenodd"
              d="M7.293 14.707a1 1 0 010-1.414L10.586 10 7.293 6.707a1 1 0 011.414-1.414l4 4a1 1 0 010 1.414l-4 4a1 1 0 01-1.414 0z"
              clip-rule="evenodd"
            /></svg
          >
        </A>
      </div>
    </div>

    <div class="p-6">
      <Label class="space-y-2 p-6">
        <span>SQLite DB file URL</span>
        <Input type="url" placeholder="" size="md" bind:value={dbUrl} />
      </Label>
      <div class="grid md:grid-cols-4 gap-4 mt-4 p-6">
        <div>
          <Label class="space-y-2"
            >Select page size
            <Select class="mt-2" items={pageSizes} bind:value={pageSize} />
          </Label>
        </div>
        <div>
          <Label class="space-y-2">
            <span>Regex for the random variable</span>
            <Input
              type="text"
              placeholder=""
              size="md"
              bind:value={radonRegex}
            />
            <P color="text-gray-300 dark:text-teal-500">{randomValue}</P>
          </Label>
        </div>
        <div>
          <Label class="space-y-2">
            <span>Test count</span>
            <Input
              type="text"
              placeholder=""
              size="md"
              bind:value={testcount}
            />
          </Label>
        </div>
        <div>
          <Label class="space-y-2">
            <span>Interval (ms)</span>
            <Input type="text" placeholder="" size="md" bind:value={interval} />
          </Label>
        </div>
      </div>
      <div class="p-6">
        <Label class="space-y-2">
          <span
            >SQL Query Template ({"{{"}var{"}}"} will be replaced with the random
            value)</span
          >
          <CodeJar bind:value={sqlQueryTemplate} syntax="sql" {highlight} />
          <P color="text-gray-300 dark:text-teal-300"
            ><code>{sampleSqlQuery}</code></P
          >
        </Label>
      </div>
      <div class="p-6">
        <Button on:click={runQueryTest}>
          {#if querying}
            <Spinner class="mr-3" size="4" color="white" /> Running Tests ...
          {:else}
            Run Test
          {/if}
        </Button>
      </div>

      {#if error}
        <div class="p-6 mt-2">
          <Alert color="red">
            <span class="font-medium">Error:</span>
            {errorMessage}
          </Alert>
        </div>
      {/if}

      {#if testing}
        {#if validResult}
          <div class="p-6 mt-2 h-64">
            <div class="chart-container">
              <LayerCake
                padding={{ top: 8, right: 10, bottom: 20, left: 25 }}
                x="id"
                y="timeTaken"
                yNice={4}
                yDomain={[0, null]}
                zDomain={[0, null]}
                data={$testResult}
              >
                <Svg>
                  <AxisX />
                  <AxisY ticks={4} />
                  <Line />
                  <Area />
                </Svg>
              </LayerCake>
            </div>
          </div>
          <Alert color="blue">
            <span class="font-medium">Time taken:</span>
            x̄ {math.mean($timeTaken).toFixed(2)}, Mdn {math
              .median($timeTaken)
              .toFixed(2)}, SD {math.std($timeTaken).toFixed(2)}
          </Alert>
          <div class="p-6 mt-2 h-64">
            <div class="chart-container">
              <LayerCake
                padding={{ top: 8, right: 10, bottom: 20, left: 25 }}
                x="id"
                y="bytesRead"
                yNice={4}
                yDomain={[0, null]}
                data={$testResult}
              >
                <Svg>
                  <AxisX />
                  <AxisY ticks={4} />
                  <Line />
                </Svg>
              </LayerCake>
            </div>
          </div>

          <Alert color="blue">
            <span class="font-medium">Bytes read:</span>
            x̄ {math.mean($bytesRead).toFixed(2)}, Mdn {math
              .median($bytesRead)
              .toFixed(2)}, SD {math.std($bytesRead).toFixed(2)}
          </Alert>

          <div class="p-4">
            <ButtonGroup>
              <Button
                color="light"
                size="xs"
                on:click={() => {
                  const blob = new Blob([PapaParse.unparse($testResult)], {
                    type: "text/csv",
                  });
                  saveAs(blob, "test-result.csv");
                }}
              >
                <Sheet />
                Download result as CSV
              </Button>
            </ButtonGroup>
          </div>
        {/if}
      {/if}
    </div>
  </div>
</main>

<svelte:head>
  <link
    href="https://cdnjs.cloudflare.com/ajax/libs/prism/1.22.0/themes/prism.min.css"
    rel="stylesheet"
  />
</svelte:head>

<style>
  .chart-container {
    width: 100%;
    height: 100%;
  }
</style>
