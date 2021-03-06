# -*- python -*-

import os
from collections import defaultdict
from lsst.pipe.base import Struct
from lsst.ci.hsc.validate import *

env = Environment(ENV=os.environ)
Execute(Mkdir(".scons"))

root = Dir('.').srcnode().abspath
AddOption("--raw", default=os.path.join(root, "raw"), help="Path to raw data")
AddOption("--repo", default=os.path.join(root, "DATA"), help="Path for data repository")
AddOption("--calib", default=os.path.join(root, "CALIB"), help="Path for calib repository")

RAW = GetOption("raw")
REPO = GetOption("repo")
CALIB = GetOption("calib")

def command(target, source, cmd):
    """Run a command and record that we ran it

    The record is in the form of a file in the ".scons" directory.
    """
    name = os.path.join(".scons", target)
    if isinstance(cmd, str):
        cmd = [cmd]
    out = env.Command(name, source, cmd + [Touch(name)])
    env.Alias(target, name)
    return out

class Data(Struct):
    """Data we can process"""
    def __init__(self, visit, ccd):
        Struct.__init__(self, visit=visit, ccd=ccd)

    @property
    def name(self):
        """Returns a suitable name for this data"""
        return "%d-%d" % (self.visit, self.ccd)

    @property
    def dataId(self):
        """Returns the dataId for this data"""
        return dict(visit=self.visit, ccd=self.ccd)

    def id(self, prefix="--id"):
        """Returns a suitable --id command-line string"""
        return "%s visit=%d ccd=%d" % (prefix, self.visit, self.ccd)

    def sfm(self, env):
        """Process this data through single frame measurement"""
        return command("sfm-" + self.name, [ingest, calib, preSfm],
                       ["processCcd.py " + REPO + " " + self.id() + " --doraise",
                        SfmValidation.makeScons(REPO, self.dataId),
                        ])

allData = {"HSC-R": [Data(903334, 16),
                     Data(903334, 22),
                     Data(903334, 23),
                     Data(903334, 100),
                     Data(903336, 17),
                     Data(903336, 24),
                     Data(903338, 18),
                     Data(903338, 25),
                     Data(903342, 4),
                     Data(903342, 10),
                     Data(903342, 100),
                     Data(903344, 0),
                     Data(903344, 5),
                     Data(903344, 11),
                     Data(903346, 1),
                     Data(903346, 6),
                     Data(903346, 12),
                     ],
           "HSC-I": [Data(903986, 16),
                     Data(903986, 22),
                     Data(903986, 23),
                     Data(903986, 100),
                     Data(904014, 1),
                     Data(904014, 6),
                     Data(904014, 12),
                     Data(903990, 18),
                     Data(903990, 25),
                     Data(904010, 4),
                     Data(904010, 10),
                     Data(904010, 100),
                     Data(903988, 16),
                     Data(903988, 17),
                     Data(903988, 23),
                     Data(903988, 24),
                     ],
          }

# Set up the data repository
mapper = env.Command(os.path.join(REPO, "_mapper"), [],
                     ["mkdir -p " + REPO,
                      "echo lsst.obs.hsc.HscMapper > " + os.path.join(REPO, "_mapper"),
                      ])
ingest = env.Command(os.path.join(REPO, "registry.sqlite3"), mapper,
                     ["ingestImages.py " + REPO + " " + RAW + "/*.fits --mode=link " +
                      "-c clobber=True register.ignore=True --doraise"] +
                     [RawValidation.makeScons(REPO, data.dataId) for
                      data in sum(allData.itervalues(), [])]
                      )
calib = env.Command(os.path.join(REPO, "CALIB"), ingest,
                    ["rm -f " + os.path.join(REPO, "CALIB"),
                     "ln -s " + CALIB + " " + os.path.join(REPO, "CALIB")] +
                     [DetrendValidation.makeScons(REPO, data.dataId) for
                      data in sum(allData.itervalues(), [])]
                     )

# Single frame measurement
preSfm = command("sfm", mapper, "processCcd.py " + REPO + " --doraise") # Workaround race on schema/config
sfm = {(data.visit, data.ccd): data.sfm(env) for data in sum(allData.itervalues(), [])}

# Create skymap
skymap = command("skymap", mapper,
                 ["makeSkyMap.py " + REPO + " -C skymap.py --doraise",
                  SkymapValidation.makeScons(REPO, {}),
                 ])

patchDataId = dict(tract=0, patch="5,4")
patchId = " ".join(("%s=%s" % (k,v) for k,v in patchDataId.iteritems()))


# Coadd construction
preDetect = command("detect", mapper, "detectCoaddSources.py " + REPO + " --doraise")
def processCoadds(filterName, dataList):
    """Generate coadds and run detection on them"""
    ident = "--id " + patchId + " filter=" + filterName
    exposures = defaultdict(list)
    for data in dataList:
        exposures[data.visit].append(data)
    warps = [command("warp-%d" % exp, [sfm[(data.visit, data.ccd)] for data in exposures[exp]] + [skymap],
                     ["makeCoaddTempExp.py " + REPO + " " + ident +
                      " " + " ".join(data.id("--selectId") for data in exposures[exp]) + " --doraise",
                      WarpValidation.makeScons(REPO, patchDataId, visit=exp, filter=filterName)]) for
             exp in exposures]
    coadd = command("coadd-" + filterName, warps,
                    ["assembleCoadd.py " + REPO + " " + ident + " " +
                     " ".join(data.id("--selectId") for data in dataList) + " --doraise",
                     CoaddValidation.makeScons(REPO, patchDataId, filter=filterName)
                     ])
    detect = command("detect-" + filterName, [coadd, preDetect],
                     ["detectCoaddSources.py " + REPO + " " + ident + " --doraise",
                      DetectionValidation.makeScons(REPO, patchDataId, filter=filterName)
                      ])
    return detect

coadds = {ff: processCoadds(ff, allData[ff]) for ff in allData}

# Multiband processing
filterList = coadds.keys()
mergeDetections = command("mergeDetections", sum(coadds.itervalues(), []),
                          ["mergeCoaddDetections.py " + REPO + " --id " + patchId + " filter=" +
                           "^".join(filterList) + " --doraise",
                           MergeDetectionsValidation.makeScons(REPO, patchDataId),
                           ])

preMeasure = command("measure", mergeDetections, "measureCoaddSources.py " + REPO + " --doraise")
def measureCoadds(filterName):
    return command("measure-" + filterName, preMeasure,
                   ["measureCoaddSources.py " + REPO + " --id " + patchId + " filter=" + filterName +
                    " --doraise",
                    MeasureValidation.makeScons(REPO, patchDataId, filter=filterName),
                    ])

measure = [measureCoadds(ff) for ff in filterList]

mergeMeasurements = command("mergeMeasurements", measure,
                            ["mergeCoaddMeasurements.py " + REPO + " --id " + patchId +
                             " filter=" + "^".join(filterList) + " --doraise",
                             MergeMeasurementsValidation.makeScons(REPO, patchDataId),
                             ])

preForced = command("forced", [mapper, mergeMeasurements], "forcedPhotCoadd.py " + REPO + " --doraise")
def forcedPhot(filterName):
    return command("forced-" + filterName, [mergeMeasurements, preForced],
                   ["forcedPhotCoadd.py " + REPO + " --id " + patchId + " filter=" + filterName +
                    " --doraise",
                    ForcedValidation.makeScons(REPO, patchDataId, filter=filterName),
                    ])

forced = [forcedPhot(ff) for ff in filterList]

env.Alias("all", forced)
Default(forced)
