_By [Stefan Verhoeven](https://orcid.org/0000-0002-5821-2060), [Faruk Diblen](https://orcid.org/0000-0002-0989-929X),
[Jurriaan H. Spaaks](https://orcid.org/0000-0002-7064-4069), [Adam Belloum](https://orcid.org/0000-0001-6306-6937), and
[Christiaan Meijer](https://orcid.org/0000-0002-5529-5761)._

# C++ web app with WebAssembly, Vega, Web Worker and React

In previous blogs in the series we learned

* [using C++ in a web app with WebAssembly](../webassembly/README.md)
* [how to perform computations without blocking the user interface](../web-worker/README.md)
* [how to interact with your C++ web app using React forms](../react/README.md)
* [how to visualize data from the algorithm](../vega/README.md)

Those topics by them selves are useful, but how can all those topics be put together into a single web application. This blog will tell you how to do build the web application including everything but the kitchen sink.

![Pack everything but the kitchen sink](1024px-Pack_Gong_(3144438149).jpg)

_Pack everything but the kitchen sink. Image courtesy of [WikiMedia Commons](https://commons.wikimedia.org/wiki/File:Pack_Gong_(3144438149).jpg)._

We want to make an React application which will find the root of an equation using Newton-Raphson algorithm and plot each iteration of the algorithm. Let us go over the pieces for this application next.

## WebAssembly module

The WebAssembly module contains the equation and Newton-Raphson algorithm. We will reuse the module made in the visualization blog post, so we will download [newtonraphson.js](https://github.com/NLESC-JCER/run-cpp-on-web/blob/master/js-plot/newtonraphson.js) and [newtonraphson.wasm](https://github.com/NLESC-JCER/run-cpp-on-web/blob/master/js-plot/newtonraphson.wasm).

## Web Worker

In the [web worker blog](TODO) we used a Web Worker thread to not block the user interface while busy with a computation. A Web Worker is not needed for the quick computation we are using, but let's be a good citizen and not claim the main thread when we don't need to.

![High five](high-five.jpg)
_UI and worker thread working together. Image from [pxhere](https://pxhere.com/en/photo/1451159)._

The work code of the Web Worker blog needs to be enhanced with returning the iterations like

```javascript
// this JavaScript snippet is stored as webassembly/worker.js
importScripts('newtonraphson.js');

// this JavaScript snippet is later referred to as <<worker-provider-onmessage>>
onmessage = function(message) {
  // this JavaScript snippet is before referred to as <<handle-message>>
  if (message.data.type === 'CALCULATE') {
    createModule().then(({NewtonRaphson}) => {
      // this JavaScript snippet is before referred to as <<perform-calc-in-worker>>
      const tolerance = message.data.payload.tolerance;
      const newtonraphson = new NewtonRaphson(tolerance);
      const initial_guess = message.data.payload.initial_guess;
      const root = newtonraphson.solve(initial_guess);
      // newtonraphson.iterations is a vector object which not consumeable by Vega
      // So convert Emscripten vector of objects to JavaScript array of objects
      const iterations = new Array(
        newtonraphson.iterations.size()
      ).fill().map(
        (_, iteration) => {
          return newtonraphson.iterations.get(iteration)
        }
      );
      // this JavaScript snippet is before referred to as <<post-result>>
      postMessage({
        type: 'RESULT',
        payload: {
          root: root,
          iterations: iterations
        }
      });
    });
  }
};
```
File: _worker.js_

## React

In the [React blog post](../react/README.md) we wrote several React components to make a form let's re-use most of it, but instead of calling the WebAssembly module directly we will use a Web Worker.

```js
const [result, setResult] = React.useState({ root: undefined, iterations: [] });

function handleSubmit(event) {
  event.preventDefault();
  const worker = new Worker('worker.js');
  worker.onmessage = function (message) {
    if (message.data.type === 'RESULT') {
      const result = message.data.payload;
      setResult(result);
      worker.terminate();
    }
  };
  worker.postMessage({
    type: 'CALCULATE',
    payload: { tolerance: tolerance, initial_guess: initial_guess }
  });
}
```

## Visualization

In the visualization [blog](../vega/README.md) post we used Vega-Lite to make 2 plots. Let's connect these 2 plots into a single visualization where zooming and panning in one plot will be reflected in other plot.

To generate the Vega-Lite specification we can write a function like so

```js
function iterations2spec(iterations) {
  // Because the initial guess can be changed now we need to scale equation line to iterations
  const max_x = Math.max(...iterations.map(d => d.x));
  const min_x = Math.min(...iterations.map(d => d.x));
  const equation_line = {
    "width": 800,
    "height": 600,
    "data": {
      "sequence": {
        // Draw equation within -4 to 4 window, if iterations are inside of window
        // otherwise use iterations min to max x
        "start": min_x > -4 ? -4 : min_x,
        "stop": max_x < 4 ? 4 : max_x,
        "step": 0.1, "as": "x"
      }
    },
    "transform": [
      {
        "calculate": "2 * datum.x * datum.x * datum.x - 4 * datum.x * datum.x + 6",
        "as": "y"
      }
    ],
    "mark": "line",
    "encoding": {
      "x": { "type": "quantitative", "field": "x" },
      "y": { "field": "y", "type": "quantitative" },
      "color": { "value": "#f00" }
    }
  };
  const root_rule = {
    "data": { "values": [{ "x": -1 }] },
    "encoding": { "x": { "field": "x", "type": "quantitative" } },
    "layer": [
      { "mark": { "type": "rule", "strokeDash": [4, 8] } },
      { "mark":
        {
          "type": "text",
          "align": "right",
          "baseline": "bottom",
          "dx": -4,
          "dy": 10,
          "x": "width",
          "fontSize": 20,
          "text": "root = -1.0"
        }
      }
    ]
  };
  const iterations_scatter = {
    "title": {
      "text": "2x^3 - 4x^2 + 6 with root",
      "fontSize": 20,
      "fontWeight": "normal"
    },
    "data": {
      "values": iterations
    },
    "encoding": {
      "x": {
        "field": "x",
        "type": "quantitative",
        "title": "x",
        "axis": {
          "labelFontSize": 20,
          "titleFontSize": 20,
          "labelFontWeight": "lighter",
          "tickMinStep": 1.0
        }
      },
      "y": {
        "field": "y",
        "type": "quantitative",
        "title": "y",
        "axis": {
          "labelFontSize": 20,
          "titleFontSize": 20,
          "labelFontWeight": "lighter",
          "titleX": -65
        }
      },
      "text": { "field": "index", "type": "nominal" }
    },
    "layer": [
      {
        "mark": { "type": "circle", "tooltip": { "content": "data" } },
        "selection": { "grid": { "type": "interval", "bind": "scales" } }
      },
      { "mark": { "type": "text", "dy": -10, "fontSize": 16 } }
    ]
  };
  const iteration_vs_y = {
    "title": {
      "text": "Iterations",
      "fontSize": 20,
      "fontWeight": "normal"
    },
    "width": 400,
    "height": 600,
    "data": {
      "values": iterations
    },
    "encoding": {
      "x": {
        "field": "index",
        "type": "quantitative",
        "title": "Iteration index",
        "axis": {
          "labelFontSize": 20,
          "titleFontSize": 20,
          "labelFontWeight": "lighter",
          "tickMinStep": 1.0
        }
      },
      "y": {
        "field": "y",
        "type": "quantitative",
        "axis": {
          "labelFontSize": 20,
          "titleFontSize": 20,
          "labelFontWeight": "lighter",
          "titleX": -65
        }
      }
    },
    "mark": {
      "type": "line",
      "point": true,
      // Enable tooltip so on mouseover it shows all data of that iteration
      "tooltip": { "content": "data" }
    },
    // Enable zooming and panning
    "selection": { "grid": { "type": "interval", "bind": "scales" } }
  };

  return {
    "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
    "hconcat": [
      {
        "layer": [
          equation_line,
          root_rule,
          iterations_scatter
        ]
      }
      , iteration_vs_y
    ]
  };
}
```

To wrap the vega visualization in React component we will use `useRef` to get a DOM element as container and use `useEffect` to call vegaEmbed when the iterations or container changes.
The React component to render the visualization is

```jsx
function IterationsPlot({ iterations }) {
  const container = React.useRef(null);

  function didUpdate() {
    if (container.current === null || iterations.length === 0) {
      return;
    }
    const spec = iterations2spec(iterations);
    vegaEmbed(container.current, spec);
  }

  const dependencies = [container, iterations];
  React.useEffect(didUpdate, dependencies);

  return <div ref={container} />;
}
```

## Pack it up

The React components and React render call can be packed up all together in a JavaScript file called [app.js](https://github.com/NLESC-JCER/run-cpp-on-web/blob/master/kitchen-sink/app.js).

The web applications needs a HTML page to fetch all the React and Vega dependencies, define a HTML tag for rendering the React app to and finally include the application JavaScript file.

```html
<!doctype html>
<html lang="en">
  <head>
    <title>Example WebAssembly/Web Worker/React/Vega-Lite application</title>
    <script src="https://unpkg.com/react@16/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@16/umd/react-dom.development.js" crossorigin></script>
    <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/vega@5.13.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/vega-lite@4.13.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/vega-embed@6.8.0"></script>
  </head>
  <div id="container"></div>
  <script type="text/babel" src="app.js"></script>
</html>
```
File: _app.html_

We'll need a web server to display the HTML page in a web browser. For this, we'll use the http.server module from Python 3 to host all files on port 8000, like so:

```shell
python3 -m http.server 8000
```

Visiting the page at [http://localhost:8000/app.html](http://localhost:8000/app.html) should give us a plot like

[![Image](app.gif)](https://github.com/NLESC-JCER/run-cpp-on-web/blob/master/kitchen-sink/app.html)

(Click on image to get interactive version)

You can try out different initial guesses to get different amount of iterations.
For example having initial guess located in a local minimum like `2` will make the algorithm use many iterations to jump over the minimum.

## Recap

In this series of blog posts we introduced a lot of different technologies to able to take an algorithm written in C++ and make a interactive web application that will run fully in a web browser.

All the source code shown is available at [https://github.com/NLESC-JCER/run-cpp-on-web](https://github.com/NLESC-JCER/run-cpp-on-web).

This blog was written by the Generalization Team of the Netherlands eScience Center. The team consists of Stefan Verhoeven, Faruk Diblen, Jurriaan H. Spaaks, Adam Belloum and Christiaan Meijer. Feel free to get in touch with the generalization team at generalization@esciencecenter.nl.

Hope you enjoyed this series of blogs and if you have suggestions or questions please post a comment below.

_These blogs were written as part of the "Passing XSAMS" project. To learn more about the project, check out its
[project page](https://www.esciencecenter.nl/projects/passing-xsams/)._
