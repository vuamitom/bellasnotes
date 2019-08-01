---
layout: post
title:  "Run Babel transpiler on React with Odoo's asset bundling"
date:   2019-07-30 00:12:33 +0700
categories: odoo
tags: [odoo,babel,react]
---

Odoo web client default framework is based on a combination of jQuery and BackboneJS. Now, jQuery is old but it is not necessary worse. Technologies serve its purpose as long as it works and meet requirements. However, there is cases when non-product requirements need to be considered such as when your development team is heavily invested on, say ReactJs, and productivity can be greatly improved when the familiar toolset is available. Needless to say, productivity means cost saving. Thus, this post quickly looks into necessary steps to integrate Babel transpiler with Odoo's asset bundling.

Web clients in Odoo have their asset bundled and minified in production. The bundler supports css preprocessors like Sass and Less right out of the box. Support for javascript transpiler like Babel was not built-in. Fortunately, supporting Babel transpiling can be achieved via sub-classing the default bundler. 

The first step is to let QWeb module knows that it should use our custom asset bundler clas. The below code inherit QWeb model to create `AssetsBundleBabel` instead of the default class.

```python

class QWeb(models.AbstractModel):
    """ QWeb object for rendering stuff in the website context """

    _inherit = 'ir.qweb'

    def get_asset_bundle(self, xmlid, files, remains=None, env=None):
        return AssetsBundleBabel(xmlid, files, remains=remains, env=env)
```

`AssetsBundleBabel` is our custom class that looks for jsx assets and pre-process them with Babel before bundling. Since there is no readily available Python binding for Babel, we will make do with a subprocess call to babel command.

```bash
# install babel command line
npm install -g babel-cli
npm install -g babel-preset-react
```
The snippet below for `AssetBundleBabel` is pretty self descriptive. There is one caveat that the inclusion order of js files in the output bundle is not maintained. If this is a big deal, further work can be done to explore running Babel on every javascript in place of Odoo's minify function. (And be prepared to deal with strict mode warnings).

```python
from odoo.addons.base.models.assetsbundle import AssetsBundle

class AssetsBundleBabel(AssetsBundle):

	def __init__(self, name, files, remains=None, env=None):
        super(AssetsBundleBabel, self).__init__(name, files, remains=remains, env=env)
        for idx, js in enumerate(self.javascripts):
            if js.url.endswith('.jsx'):
            	# create a custom asset object for jsx files
                self.javascripts[idx] = BabelJavascriptAsset(self, url=js.url, filename=js._filename, inline=js.inline)

    def transpile_babel(self):
        """run babel to transpile jsx and minify"""
        if self.javascripts:
            need_transpile = [asset for asset in self.javascripts if isinstance(asset, BabelJavascriptAsset)]
            compiled = ''
            if len(need_transpile) > 0:
                source = '\n'.join([asset.content for asset in need_transpile])
                compiler = need_transpile[0].compile 
                try:
                    compiled = compiler(source)
                    return compiled
                except CompileError as e:
                    return handle_compile_error(e, source=source)
            else:
                return compiled

    def js(self):
        # to override to run transpiler
        attachments = self.get_attachments('js')
        if not attachments:
        	# minify normal javascript as usual.
            content = ';\n'.join([asset.minify() for asset in self.javascripts if not isinstance(asset, BabelJavascriptAsset)])      
            # append transpiled js at the end
            # this part needs improvement      
            content = content + ';\n' + self.transpile_babel()
            return self.save_attachment('js', content)
        return attachments[0] 
```

Notice the `BabelJavascriptAsset` class, which is in charge of calling babel commandline.

```python
class BabelJavascriptAsset(JavascriptAsset):
    """TODO: override content"""
    def get_command(self):
        return ['babel', '--presets=/usr/lib/node_modules/babel-preset-react']

    def compile(self, source):
        command = self.get_command()
        try:
            compiler = Popen(command, stdin=PIPE, stdout=PIPE,
                             stderr=PIPE)
        except Exception:
            raise CompileError("Could not execute command %r" % command[0])

        (out, err) = compiler.communicate(input=source.encode('utf-8'))
        if compiler.returncode:
            cmd_output = misc.ustr(out) + misc.ustr(err)
            if not cmd_output:
                cmd_output = u"Process exited with return code %d\n" % compiler.returncode
            raise CompileError(cmd_output)
        return out.decode('utf8')
```

After the above steps, you can use jsx with your Odoo asset bundle. 