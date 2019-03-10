fs = require 'fs'
glob = require 'glob'
os = require 'os'
path = require 'path'
util = require 'util'

Browserify = require 'browserify'
CoffeeLint = require 'coffeelint'
CoffeeScript = require 'coffeescript'
Mocha = require 'mocha'
less = require 'less'

Promisified =
  copyFile: util.promisify fs.copyFile
  glob: util.promisify glob
  mkdir: util.promisify fs.mkdir
  readFile: util.promisify fs.readFile
  readdir: util.promisify fs.readdir
  stat: util.promisify fs.stat
  writeFile: util.promisify fs.writeFile

SRC = path.join __dirname, 'src'
LIB = path.join __dirname, 'lib'
TEST = path.join __dirname, 'test'
TEMPLATE_SRC = path.join __dirname, 'templates'
TEMPLATE_OUTPUT = path.join SRC, 'templates.coffee'

# HELPERS

ensureDir = (p) ->
  return new Promise (resolve, reject) ->
    return Promisified.stat(p).then(
      (st) ->
        if st.isDirectory()
          return resolve()
        else
          return reject(new Error("#{p} exists, but not a directory"))
    ).catch () ->
      return Promisified.mkdir(p).then(resolve).catch(reject)

replaceExt = (filePath, newExt) ->
  filePath.substr(0, filePath.lastIndexOf('.')) + newExt

recursiveFindExt = (p, extension) ->
  return Promisified.stat(p).then(
    (st) ->
      if st.isDirectory()
        return Promisified.glob(
          "#{p}/**/*#{extension}",
          ignore: 'node_modules/**'
        ).then(
          (files) ->
            files.concat(files)
            return Promise.resolve files.map (p) -> path.join p
        )
      else
        return Promise.reject(new Error("#{p} is not a directory"))
  )

compileCoffeeFile = (f, srcDir = SRC, destDir = LIB) ->
  outfile = path.join(
    destDir, path.relative(srcDir, replaceExt(f, '.js'))
  )
  return Promisified.readFile(f, 'utf-8').then(
    (data) ->
      return Promisified.writeFile(outfile, CoffeeScript.compile(data)).then(
        () -> return Promise.resolve(outfile)
      )
  )

lintCoffeeFile = (f) ->
  return Promisified.readFile(f, 'utf-8').then (data) ->
    return Promise.resolve
      file: f,
      errors: CoffeeLint.lint data

compileLessFile = (f, srcDir = SRC, destDir = LIB) ->
  outfile = path.join(
    destDir, path.relative(
      srcDir,
      replaceExt(f, '.css')
    )
  )
  return Promisified.readFile(f, 'utf-8').then(
    (data) ->
      return less.render(data).then(
        (result) -> return Promisified.writeFile(outfile, result.css).then(
          () -> return Promise.resolve(outfile)
        )
      )
  )

exportify = (f) ->
  templateName = replaceExt f, ''
  templateExportName = templateName.replace '-', '.'
  templateFilePath = "#{ TEMPLATE_SRC }/#{ f }"
  return Promisified.readFile(
    templateFilePath, 'utf-8'
  ).then(
    (body) ->
      return Promise.resolve(
        "exports.#{ templateExportName } = \"\"\"#{ body }\"\"\""
      )
  )

copyFileIfNotExists = (src, dest) ->
  return Promisified.copyFile(src, dest, fs.constants.COPYFILE_EXCL).then(
    () -> return Promise.resolve()
  ).catch (err) ->
    if err.code is 'EEXIST'
      return Promise.resolve()
    else
      return Promise.reject err

# TASKS

compileTemplates = () ->
  console.log 'Compiling templates/*.html to src/templates.coffee...'
  return Promisified.readdir(TEMPLATE_SRC).then(
    (files) ->
      return Promise.all(
        exportify f for f in files
      ).then(
        (templateBlocks) ->
          content = (
            '# TEMPLATES.COFFEE IS AUTO-GENERATED. CHANGES WILL BE LOST!\n'
          )
          content += templateBlocks.join '\n\n'
          return Promisified.writeFile(TEMPLATE_OUTPUT, content, 'utf-8')
      )
  )

compileCoffee = () ->
  console.log 'Compiling src/*.coffee to lib/*.js...'
  return recursiveFindExt(SRC, '.coffee').then(
    (files) ->
      return Promise.all(compileCoffeeFile(f) for f in files)
  )

compileLess = () ->
  console.log 'Compiling src/*.less to lib/*.css...'
  return recursiveFindExt(SRC, '.less').then(
    (files) ->
      return Promise.all(compileLessFile(f) for f in files)
  )

browserify = () ->
  console.log 'Browserifying lib/{client,browser}.js to lib/log.io.js...'
  return new Promise (resolve, reject) ->
    b = Browserify()
    b.add [
      path.join(LIB, 'client.js'),
      path.join(LIB, 'browser.js')
    ]
    b.bundle().pipe(
      fs.createWriteStream(path.join(LIB, 'log.io.js'))
    ).on(
      'finish', resolve
    ).on(
      'error', reject
    )

lint = () ->
  console.log 'Linting *.coffee and Cakefile...'
  return Promise.all(
    [
      recursiveFindExt(SRC, '.coffee'),
      recursiveFindExt(TEST, '.coffee')
    ]
  ).then(
    (results) ->
      files = [].concat.apply([], results)
      files = files.filter (f) -> f != TEMPLATE_OUTPUT
      files.push(__filename)
      return Promise.all(lintCoffeeFile(f) for f in files).then(
        (results) ->
          errorCount = 0
          for result in results
            errorCount += result.errors.length
            for error in result.errors
              message = "#{result.file}:#{error.lineNumber} #{error.message}"
              if error.context
                message += ": #{error.context}"
              console.error message
          if errorCount > 0
            s = if errorCount > 1 then 's' else ''
            return Promise.reject(
              new Error "CoffeeLint reported #{errorCount} error#{s}"
            )
          else
            return Promise.resolve()
      )
  )

test = () ->
  console.log 'Running tests...'
  return new Promise (resolve, reject) ->
    return compileCoffeeFile(
      path.join(TEST, 'functional.coffee'), TEST, TEST
    ).then(
      (js) ->
        mocha = new Mocha
        mocha.addFile(js)
        mocha.run (errorCount) ->
          if errorCount > 0
            s = if errorCount > 1 then 's' else ''
            return reject(
              new Error "Mocha reported #{errorCount} error#{s}"
            )
          else
            return resolve()
    )

ensureConfig = () ->
  console.log 'Ensuring configuration files exist under ~/.log.io/...'
  confDir = path.join os.homedir(), '.log.io'
  return ensureDir(confDir).then(
    () ->
      return Promise.all(
        copyFileIfNotExists(
          path.join(__dirname, 'conf', c + '.conf'),
          path.join(confDir, c + '.conf')
        ) for c in ['harvester', 'log_server', 'web_server']
      )
  ).catch (err) ->
    console.error 'Failed to ensure configuration file existence!'
    console.error 'Try running npm as a specific user:'
    console.error '  npm install -g log.io --user <username>'
    return Promise.reject err

task(
  'compile:templates',
  'Compile templates/*.html to src/templates.coffee',
  ->
    compileTemplates().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'compile:coffee',
  'Compile *.coffee to *.js',
  ->
    compileCoffee().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'compile:less',
  'Compile *.less to *.css',
  ->
    compileLess().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'browserify',
  'Browserify lib/client.js to lib/log.io.js',
  ->
    browserify().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'lint',
  'Lint *.coffee and Cakefile',
  ->
    lint().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'test',
  'Run functional tests',
  ->
    test().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'ensure:config',
  'Ensure that config files exist in ~/.log.io/',
  ->
    ensureConfig().catch (err) ->
      console.error err.message
      process.exit(1)
)

task(
  'build',
  'Build Log.io package',
  ->
    console.log 'Building Log.io package...'
    ensureDir(LIB).then(
      () -> return compileTemplates()
    ).then(
      () -> return compileCoffee()
    ).then(
      () -> return compileLess()
    ).then(
      () -> return browserify()
    ).catch(
      (err) ->
        console.error err.message
        process.exit(1)
    )
)
