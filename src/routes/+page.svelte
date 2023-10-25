<script lang="ts">
  import './style.scss';
  import { onMount, onDestroy } from 'svelte';
  import svgo from 'svgo';

  let svgs: [string, File][] = [];
  let out: [string, string][] = [];

  $: {
    out = [];
    if (typeof DOMParser !== 'undefined') svgs.forEach(handleSvg);
  }
  const handleSvg = ([content, file]: [string, File]) => {
    const parser = new DOMParser();
    try {
      const doc = parser.parseFromString(content, 'application/xml');
      const errorNode = doc.querySelector('parsererror');
      if (errorNode) {
        console.error(errorNode);
        throw new Error('error while parsing');
      } else {
        const layers = [...(doc.documentElement.children as any)].filter(v => {
          return v.getAttribute('inkscape:groupmode') === 'layer';
        });
        if (layers.length === 0)
          return alert(
            'This is not a valid inkscape export, or has been modified/optimized prior to being dragged in here. Throw it in inkscape first, make sure the root has one or more layers, then try again.',
          );
        const displayRegex = /display ?: ?none(?: !important)?;?/giu;
        layers.forEach(layer => {
          layer.setAttribute(
            'style',
            'display:none !important;' +
              (layer.getAttribute('style')?.replace(displayRegex, '') ?? ''),
          );
        });
        layers.forEach((layer, idx) => {
          const layerName =
            layer.getAttribute('inkscape:label') ?? `Layer ${idx}`;
          const noneDisplayStyle = layer.getAttribute('style')!;
          const noNoneDisplayStyle = noneDisplayStyle.replace(displayRegex, '');
          layer.setAttribute('style', noNoneDisplayStyle);
          const svgRaw = doc.documentElement.outerHTML;
          layer.setAttribute('style', noneDisplayStyle);
          const svgOut = svgo.optimize(svgRaw, {
            plugins: [
              {
                name: 'preset-default',
                params: {
                  overrides: {
                    removeViewBox: false,
                  },
                },
              },
              'collapseGroups',
              'convertEllipseToCircle',
              'convertPathData',
              {
                name: 'prefixIds',
                params: {
                  prefix: Math.random().toString(36).slice(2),
                },
              },
            ],
            floatPrecision: 3,
          });
          const svg = svgOut.data;
          const dataUrl = `data:${file.type};base64,${btoa(svg)}`;
          const fileBaseName = file.name.replace(/\.svg/giu, '');
          fetch(dataUrl)
            .then(v => v.blob())
            .then(v => URL.createObjectURL(v))
            .then(blob => {
              out = [
                ...out,
                [
                  blob,
                  `${`${fileBaseName}-${layerName}`
                    .toLowerCase()
                    .replace(/ /g, '-')
                    .replace(/[^\w-]+/g, '')}.svg`,
                ],
              ];
            });
        });
      }
    } catch (error) {
      console.error(error);
      return alert(
        'Encountered an error (see console). Make sure the file is valid, has one or more layers, and was exported from Inkscape.',
      );
    }
  };

  const displayFileContents = (files: File[]) => {
    svgs = [];
    for (const file of files) {
      const reader = new FileReader();
      reader.onload = e => {
        const content = e.target?.result;
        if (!content) return console.warn('No content.');
        if (typeof content !== 'string')
          return alert(
            'Got ' +
              typeof content +
              ', expected string. Is this a binary file?',
          );
        svgs = [...svgs, [content, file]];
      };
      reader.readAsText(file);
    }
  };
  let dropArea: HTMLDivElement;
  let fileInput: HTMLInputElement;
  let listening = true;
  let dropping = false;
  onMount(() => {
    if (!dropArea || !fileInput) throw new Error('missing el');

    const els = [dropArea, document as unknown as HTMLElement];

    // Prevent default drag behaviors
    ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
      if (!listening) return;
      dropArea.addEventListener(eventName, e => {
        e.preventDefault();
        e.stopPropagation();
      });
    });

    // Highlight drop area when a file is dragged over
    ['dragenter', 'dragover'].forEach(eventName => {
      if (!listening) return;
      els.forEach(v =>
        v.addEventListener(eventName, () => {
          dropping = true;
        }),
      );
    });

    ['dragleave', 'drop'].forEach(eventName => {
      if (!listening) return;
      document.addEventListener(eventName, () => {
        dropping = false;
      });
    });

    // Handle dropped files
    els.forEach(v =>
      v.addEventListener('drop', e => {
        if (!listening) return;
        e.preventDefault();
        e.stopPropagation();
        const files = e.dataTransfer?.files;
        if (!files) throw new Error('no files');
        displayFileContents(files as unknown as File[]);
      }),
    );

    // Handle file input change (fallback for selecting files)
    fileInput.addEventListener('change', () => {
      if (!listening) return;
      const files = (fileInput as any).files;
      if (!files) throw new Error('no files');
      displayFileContents(files);
    });
  });
  onDestroy(() => {
    listening = false;
  });
  const dlAll = () =>
    document.querySelectorAll('a.icon').forEach(v => (v as any).click());
</script>

<svelte:head>
  <title>Inklayer</title>
  <meta
    name="description"
    content="Extract all layers from an Inkscape SVG automatically."
  />
</svelte:head>

<div
  class="dropArea"
  bind:this={dropArea}
  style={out.length === 0
    ? dropping
      ? 'border-color: #88aade;background: #ffffff0a;'
      : ''
    : 'display:none;'}
>
  <h2>{dropping ? 'drop the file' : 'upload a file'}</h2>
  <button on:click={() => fileInput.click()}>Upload</button>
  <input
    type="file"
    bind:this={fileInput}
    style="opacity:0;position:fixed;top:0;left:0;"
  />
</div>

{#if out.length > 0}
  <h2>layers</h2>
  <div class="icons">
    {#each out as svg}
      <a href={svg[0]} download={svg[1]} title={svg[1]} class="icon">
        <img src={svg[0]} alt={svg[1]} />
      </a>
    {/each}
  </div>
  <h2>actions</h2>
  <div class="btns" style={dropping ? 'border-color: #88aade;' : ''}>
    <button on:click={dlAll}>Download All</button>
    <button on:click={() => (out = [])}
      >{dropping ? 'Drop the file' : 'Upload Again'}</button
    >
  </div>
{/if}

<style lang="scss">
  h2 {
    margin-top: 12px;
    margin-bottom: 4px;
    text-align: center;
  }
  .dropArea {
    border: 2px solid #88aade00;
    transition: 0.5s;
    padding: 24px;
    border-radius: 12px;
    flex-direction: column;
    h2:first-child {
      margin-top: 0;
    }
  }
  .icons a img {
    max-width: 128px;
    max-height: 128px;
  }
  .icons,
  .btns,
  .dropArea {
    max-width: calc(100vw - 16px);
    display: flex;
    align-items: center;
    justify-content: center;
    flex-wrap: wrap;
    gap: 4px;
  }
  .icons {
    flex-direction: row;
  }
  .btns {
    border: 2px solid #88aade00;
    flex-direction: row;
    margin-top: 4px;
    padding: 12px;
    box-shadow: var(--sclr, #0006) var(--distX, 4px) var(--distY, 4px)
      var(--blur, 8px);
    background: #fff1;
    border-radius: 12px;
    transition: 0.4s;
    &:hover {
      --sclr: #0007;
      --distX: 6px;
      --distY: 6px;
      --blur: 12px;
    }
  }
</style>
