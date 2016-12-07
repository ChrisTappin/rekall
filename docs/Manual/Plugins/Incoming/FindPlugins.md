---
abstract: Find which plugin(s) are available to produce the desired output.
args: {producers_only: 'Only include producers: plugins that output only this struct
    and have no side effects. (type: Boolean)

    ', type_name: 'The name of the type we''re looking for. E.g.: ''proc'' will find
    psxview, pslist, etc. (type: String)

    ', verbosity: 'An integer reflecting the amount of desired output: 0 = quiet,
    10 = noisy. (type: IntParser)



    * Default: 1'}
class_name: FindPlugins
epydoc: rekall.plugins.common.efilter_plugins.search.FindPlugins-class.html
layout: plugin
module: rekall.plugins.common.efilter_plugins.search
title: which_plugin
---