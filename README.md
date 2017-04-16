# api documentation for  [onchange (v3.2.1)](https://github.com/Qard/onchange)  [![npm package](https://img.shields.io/npm/v/npmdoc-onchange.svg?style=flat-square)](https://www.npmjs.org/package/npmdoc-onchange) [![travis-ci.org build-status](https://api.travis-ci.org/npmdoc/node-npmdoc-onchange.svg)](https://travis-ci.org/npmdoc/node-npmdoc-onchange)
#### Use glob patterns to watch file sets and run a command when anything is added, changed or deleted.

[![NPM](https://nodei.co/npm/onchange.png?downloads=true&downloadRank=true&stars=true)](https://www.npmjs.com/package/onchange)

[![apidoc](https://npmdoc.github.io/node-npmdoc-onchange/build/screenCapture.buildCi.browser.%252Ftmp%252Fbuild%252Fapidoc.html.png)](https://npmdoc.github.io/node-npmdoc-onchange/build/apidoc.html)

![npmPackageListing](https://npmdoc.github.io/node-npmdoc-onchange/build/screenCapture.npmPackageListing.svg)

![npmPackageDependencyTree](https://npmdoc.github.io/node-npmdoc-onchange/build/screenCapture.npmPackageDependencyTree.svg)



# package.json

```json

{
    "author": {
        "name": "Stephen Belanger"
    },
    "bin": {
        "onchange": "cli.js"
    },
    "bugs": {
        "url": "https://github.com/Qard/onchange/issues"
    },
    "dependencies": {
        "arrify": "~1.0.1",
        "chokidar": "~1.6.0",
        "cross-spawn": "~4.0.0",
        "minimist": "~1.2.0",
        "tree-kill": "~1.1.0"
    },
    "description": "Use glob patterns to watch file sets and run a command when anything is added, changed or deleted.",
    "devDependencies": {},
    "directories": {},
    "dist": {
        "shasum": "7669312c8a8f94d80b4595dc8abe5d119559f7e0",
        "tarball": "https://registry.npmjs.org/onchange/-/onchange-3.2.1.tgz"
    },
    "gitHead": "b42975da41c899feda42ced7fe1d413aa297c3e1",
    "homepage": "https://github.com/Qard/onchange",
    "keywords": [
        "glob",
        "watch",
        "change"
    ],
    "license": "MIT",
    "main": "index.js",
    "maintainers": [
        {
            "name": "blakeembrey"
        },
        {
            "name": "qard"
        }
    ],
    "name": "onchange",
    "optionalDependencies": {},
    "repository": {
        "type": "git",
        "url": "git+https://github.com/Qard/onchange.git"
    },
    "scripts": {
        "test": "node test/index.js"
    },
    "typings": "index.d.ts",
    "version": "3.2.1"
}
```



# <a name="apidoc.tableOfContents"></a>[table of contents](#apidoc.tableOfContents)

#### [module onchange](#apidoc.module.onchange)
1.  [function <span class="apidocSignatureSpan"></span>onchange (match, command, args, opts)](#apidoc.element.onchange.onchange)
1.  [function <span class="apidocSignatureSpan">onchange.</span>toString ()](#apidoc.element.onchange.toString)



# <a name="apidoc.module.onchange"></a>[module onchange](#apidoc.module.onchange)

#### <a name="apidoc.element.onchange.onchange"></a>[function <span class="apidocSignatureSpan"></span>onchange (match, command, args, opts)](#apidoc.element.onchange.onchange)
- description and source-code
```javascript
onchange = function (match, command, args, opts) {
  opts = opts || {}

  var matches = arrify(match)
  var initial = !!opts.initial
  var wait = !!opts.wait
  var cwd = opts.cwd ? resolve(opts.cwd) : process.cwd()
  var stdout = opts.stdout || process.stdout
  var stderr = opts.stderr || process.stderr
  var delay = Number(opts.delay) || 0
  var killSignal = opts.killSignal || 'SIGTERM'
  var outpipe = typeof opts.outpipe === 'string' ? outpipetmpl(opts.outpipe) : undefined

  if (!command && !outpipe) {
    throw new TypeError('Expected "command" and/or "outpipe" to be specified')
  }

  var childOutpipe
  var childCommand
  var pendingOpts
  var pendingTimeout
  var pendingExit = false

  // Convert arguments to templates
  var tmpls = args ? args.map(tmpl) : []
  var watcher = chokidar.watch(matches, {
    cwd: cwd,
    ignored: opts.exclude || [],
    usePolling: opts.poll === true || typeof opts.poll === 'number',
    interval: typeof opts.poll === 'number' ? opts.poll : undefined
  })

  // Logging
  var log = opts.verbose ? function log (message) {
    stdout.write('onchange: ' + message + '\n')
  } : function () {}

<span class="apidocCodeCommentSpan">  /**
   * Run when the script exits.
   */
</span>  function onexit () {
    if (childOutpipe || childCommand) {
      return
    }

    pendingExit = false

    if (pendingOpts) {
      if (pendingTimeout) {
        clearTimeout(pendingTimeout)
      }

      if (delay > 0) {
        pendingTimeout = setTimeout(function () {
          cleanstart(pendingOpts)
        }, delay)
      } else {
        cleanstart(pendingOpts)
      }
    }
  }

  /**
   * Run on fresh start (after exists, clears pending args).
   */
  function cleanstart (args) {
    pendingOpts = null
    pendingTimeout = null

    return start(args)
  }

  /**
   * Start the script.
   */
  function start (opts) {
    // Set pending options for next execution.
    if (childOutpipe || childCommand) {
      pendingOpts = opts

      if (!pendingExit) {
        if (wait) {
          log('waiting for process and restarting')
        } else {
          if (childCommand) {
            log('killing command ' + childCommand.pid + ' and restarting')
            kill(childCommand.pid, killSignal)
          }

          if (childOutpipe) {
            log('killing outpipe ' + childOutpipe.pid + ' and restarting')
            kill(childOutpipe.pid, killSignal)
          }
        }

        pendingExit = true
      }
    }

    if (pendingTimeout || pendingOpts) {
      return
    }

    if (outpipe) {
      var filtered = outpipe(opts)

      log('executing outpipe "' + filtered + '"')

      childOutpipe = exec(filtered, {
        cwd: cwd
      })

      // Must pipe stdout and stderr.
      childOutpipe.stdout.pipe(stdout)
      childOutpipe.stderr.pipe(stderr)

      childOutpipe.on('exit', function (code, signal) {
        log('outpipe ' + (code == null ? 'exited with ' + signal : 'completed with code ' + code))

        childOutpipe = null

        return onexit()
      })
    }

    if (command) {
      // Generate argument strings from templates.
      var filtered = tmpls.map(function (tmpl) {
        return tmpl(opts)
      })

      log('executing command "' + [command].concat(filtered).join(' ') + '"')

      childCommand = spawn(command, filtered, {
        cwd: cwd,
        stdio: ['ignore', childOutpipe ? childOutpipe.stdin : stdout, stderr]
      })

      childCommand.on('exit', function (code, signal) {
        log('command ' + (code == null ? 'exited with ' + signal : 'completed with code ' + code))

        childCommand = null

        return onexit()
      })
    } else {
      // No data to write to 'outpipe'.
      childOutpipe.stdin.end()
    }
  }

  watcher.on('ready', function () {
    log('watching ' + matches.join(', '))

    // Execute initial event, without changed options.
    if (initial) {
      start({ event: '', changed: '' })
    }

    // For any change, creation or deletion, try to run.
    // Restart if the last run is still active.
    watcher.on('all', function (event, changed) {
      // Log the event and the file affec ...
```
- example usage
```shell
n/a
```

#### <a name="apidoc.element.onchange.toString"></a>[function <span class="apidocSignatureSpan">onchange.</span>toString ()](#apidoc.element.onchange.toString)
- description and source-code
```javascript
toString = function () {
    return toString;
}
```
- example usage
```shell
n/a
```



# misc
- this document was created with [utility2](https://github.com/kaizhu256/node-utility2)
