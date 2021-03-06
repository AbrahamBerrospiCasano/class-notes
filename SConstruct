import functools
import os
import re

# class with helper methods
class Helper(object):
  # check if a dependency is installed, fail if not (if desired)
  @staticmethod
  def checkProgramInstalled(env, program, fail=False):
    if env.GetOption("help") or env.GetOption("clean"): return True
    conf = Configure(env, log_file=None)
    found = (conf.CheckProg(program) is not None)
    conf.Finish()

    if (not found) and fail:
      raise RuntimeError("Program \"{}\" required, but not found in PATH.".format(program))

    return found

  @staticmethod
  def translateSrcToBuild(path):
    if isinstance(path, (list, tuple)):
      return [Helper.translateSrcToBuild(x) for x in path]
    else:
      newPath = path.abspath.replace("/src", "/build")
      return (Dir(newPath) if path.isdir() else File(newPath))

  @staticmethod
  def buildMathJaxTex(dirNames, target, source, env):
    mathJaxTex = ""

    for x in source:
      with open(x.abspath, "r") as f: tex = f.read()
      commands = re.findall(r"^\s*% class-notes: begin-mathjax-commands$(.*?)" +
          r"^\s*% class-notes: end-mathjax-commands$", tex, flags=re.DOTALL | re.MULTILINE)
      mathJaxTex += "\n\n".join(commands)

    mathJaxTex = mathJaxTex.replace("\hbadness=10000", "")
    mathJaxTex = re.sub(r"^\s*%.*", "", mathJaxTex, flags=re.MULTILINE)
    mathJaxTex = re.sub("\n\n+", "\n", mathJaxTex)

    if mathJaxTex != "":
      mathJaxTex = re.sub(r"(\\(?:re)?newcommand)\*", r"\1", mathJaxTex)
      mathJaxTex = re.sub(r"^(.+)$", r"  \\CustomizeMathJax{\1}", mathJaxTex, flags=re.MULTILINE)
      mathJaxTex = f"\\begin{{warpMathJax}}{mathJaxTex}\\end{{warpMathJax}}\n"

    for x in target:
      with open(x.abspath, "w") as f: f.write(mathJaxTex)

  @staticmethod
  def buildCollectionIncludeTex(dirNames, target, source, env):
    dirNames = [x for x in dirNames if x != "collection"]
    collectionIncludeTex = ""

    for dirName in dirNames:
      dirPath = f"../lectures/{dirName}/"
      collectionIncludeTex += f"""
{{%
  \subimport{{{dirPath}}}{{metadata}}%
  \input{{part}}%
  \subimport{{{dirPath}}}{{include}}%
}}
"""

    for x in target:
      with open(x.abspath, "w") as f: f.write(collectionIncludeTex)

# check SCons version
EnsureSConsVersion(3, 0)

# build flags
variables = Variables(None, ARGUMENTS)

# set up environment, export environment variables of the shell
# (for example needed for custom TeX installations which need PATH)
env = Environment(variables=variables, ENV=os.environ)

# display help about build flags when called with -h
env.Help(variables.GenerateHelpText(env))

# use timestamp to decide if a file should be rebuilt
# (otherwise SCons won't rebuild even if it is necessary)
env.Decider("timestamp-newer")

# increase line length of compiler error log lines by
# setting environment variables
env["ENV"]["max_print_line"] = "10000"
env["ENV"]["error_line"] = "254"
env["ENV"]["half_error_line"] = "238"

# use LuaTeX-based compiler
for compiler in ["luahbtex", "lualatex"]:
  if Helper.checkProgramInstalled(env, compiler, fail=True): break
  compiler = None

if compiler is None: raise RuntimeError("No suitable compiler has been found")

# filter the compiler output with custom script
compilerCommand = "python3 {} --ignore-texinputs {}".format(
    File("#/tools/filterOutput.py").abspath, compiler)
env.Replace(PDFTEX=compilerCommand, PDFLATEX=compilerCommand)

# set program name to use LaTeX instead of TeX
env.Append(PDFLATEXFLAGS="--progname=lualatex")
# show file and line number for errors
env.Append(PDFLATEXFLAGS="--file-line-error")
# enable SyncTeX for GUI editors
env.Append(PDFLATEXFLAGS="--synctex=1")
# enable shell commands
#env.Append(PDFLATEXFLAGS="--shell-escape")

# compile in build directory
for rootDirName, dirNames, fileNames in os.walk("src", followlinks=True):
  dirNames.sort()

  for fileName in sorted(fileNames):
    file_ = File(os.path.join(rootDirName, fileName))
    env.InstallAs(target=Helper.translateSrcToBuild(file_), source=file_)

# reference list to order the names of the lectures that actually exist
dirNameOrder = [
      "analysis-1",
      "analysis-2",
      "analysis-3",
      "analysis-4",
      "functional-analysis-1",
      "functional-analysis-2",
      "linear-algebra-and-analytical-geometry-1",
      "linear-algebra-and-analytical-geometry-2",
      "algebra",
      "topology",
      "probability-theory",
      "mathematical-statistics",
      "linear-control-theory",
      "numerical-linear-algebra",
      "numerical-analysis-1",
      "numerical-analysis-2",
      "partial-differential-equations",
      "approximation-and-geometric-modeling",
      "finite-elements",
      "programming-and-software-engineering",
      "data-structures-and-algorithms",
      "formal-languages-and-automata",
      "computability-and-complexity",
      "algorithmic-geometry",
      "discrete-optimization",
      "cryptographic-procedures",
      "visual-computing",
      "modeling-and-simulation",
      "optical-phenomena",
      "basic-principles-of-geosciences",
      "history-of-wind-energy-use",
    ]

# list of targets (collection must be at the end)
dirNames = ([x.name for x in Glob("src/lectures/*") if x.isdir()] + ["collection"])
dirNames.sort(key=lambda x: ((dirNameOrder.index(x), 0) if x in dirNameOrder else
      (len(dirNameOrder), x)))

for dirName in dirNames:
  dirType = ("lectures" if dirName != "collection" else "collection")
  dirPath = (f"lectures/{dirName}" if dirType != "collection" else "collection")

  dependencies = []

  # depend on all files in lecture directory
  dependencies.extend(Helper.translateSrcToBuild(Glob(f"src/{dirPath}/*")))

  # copy all files from common to lecture directory, rename preamble to <dirName>.tex
  dependencies.extend(env.Install(target=f"build/{dirPath}", source=Glob("build/common/*")))
  dependencies.extend(env.Command(target=f"build/{dirPath}/{dirName}.tex",
      source=f"build/{dirPath}/preamble.tex", action=Move("$TARGET", "$SOURCE")))
  dependencies = [x for x in dependencies if x.name != "preamble.tex"]

  if dirType == "lectures":
    # copy document.tex from parent directory to lecture directory
    dependencies.extend(env.Install(target=f"build/{dirPath}",
        source=f"build/{dirType}/document.tex"))
    # generate mathJax.tex from magic comments (copy and wrap everything between
    # "% class-notes: begin-mathjax-commands" and "% class-notes: end-mathjax-commands")
    dependencies.extend(env.Command(target=f"build/{dirPath}/mathJax.tex",
        source=Helper.translateSrcToBuild(Glob(f"src/lectures/{dirName}/*.tex")) +
        Helper.translateSrcToBuild(Glob(f"src/common/*.tex")),
        action=functools.partial(Helper.buildMathJaxTex, [dirName])))
  else:
    # generate collectionInclude.tex (including the different lectures) from the list of
    # available lectures
    dependencies.extend(env.Command(
        target=f"build/{dirPath}/collectionInclude.tex", source="",
        action=functools.partial(Helper.buildCollectionIncludeTex, dirNames)))
    # depend on source files of all lectures
    dependencies.extend(Helper.translateSrcToBuild(Glob("src/lectures/*/*")))

  # generate lecture PDF, add dependencies, create aliases
  pdf = env.PDF(target=f"build/{dirPath}/{dirName}.pdf",
      source=f"build/{dirPath}/{dirName}.tex")
  env.Depends(target=pdf, dependency=dependencies)
  env.Alias(dirName, pdf)
  env.Alias(f"{dirName}-pdf", pdf)

  # generate lecture HTML, make it depend on the PDF, create alias
  if dirType == "lectures":
    html = env.Command(target=f"build/{dirPath}/{dirName}.html",
        source=f"build/{dirPath}/{dirName}.tex",
        action=f"cd 'build/{dirPath}' && lwarpmk html && lwarpmk limages")
    env.Depends(target=html, dependency=pdf)
    env.Alias(f"{dirName}-html", html)

  # clean build subdirectory with `scons -c <dirName>`
  env.Clean("all", f"build/{dirName}")

# compile all lectures if no target is specified
env.Alias("all", "all-pdf")
env.Alias("all-pdf", [f"{x}-pdf" for x in dirNames])
env.Alias("all-html", [f"{x}-html" for x in dirNames if x != "collection"])
env.Default("all")

# clean whole build directory with `scons -c`
if GetOption("clean"): env.Clean("all", "build")

# print all available lectures
if env.GetOption("help"):
  print("")
  print("List of available lectures:")
  for dirName in dirNames: print(f"- {dirName}")
  print("")
  print("To build a specific list of lectures, append their names separated by space to the "
      "`scons` command. `scons` without any lectures will build everything.")
  print("")
