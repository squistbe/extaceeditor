Ext.define('MC.view.form.field.ExpressionEditor', {
  extend: 'Ext.form.FieldContainer',
  alternateClassName: 'ExpressionEditor',
  alias: 'widget.expressioneditor',

  DEFAULT_EDITOR_THEME: 'ace/theme/ambiance',
  EXPRESSION_VARIABLES: [
    {name: '$UserId', value: '$UserId', meta: 'variable'},
    {name: '$CurrentUser', value: '$CurrentUser', meta: 'variable'},
    {name: '$CurrentDateTime', value: '$CurrentDateTime', meta: 'variable'}
  ],

  languageTools: ace.require("ace/ext/language_tools"),
  showLineNumbers: true,
  msgTarget: 'side',
  showGutter: true,
  markerIds: [],
  gutterDecor: [],
  mixins: {
    field: 'Ext.form.field.Field'
  },

  initComponent: function () {
    var defaultItems = [],
      editor = {
        xtype: 'container',
        border: true,
        itemId: 'expression-editor',
        height: 96,
        width: '100%'
      },
      statusBar = {
        xtype: 'container',
        layout: 'hbox',
        itemId: 'statusBar',
        cls: 'expression-editor-status-bar',
        width: this.editorWidth,
        items: [
          {
            xtype: 'box',
            itemId: 'status',
            flex: 5
          },
          {
            xtype: 'box',
            itemId: 'cursorPosition',
            html: '1:1',
            style: {
              textAlign: 'right'
            },
            flex: 1
          }
        ]
      };

    if (this.editorHeight) {
      Ext.apply(editor, {
        height: this.editorHeight
      });
    } else if (this.editorRows) {
      Ext.apply(editor, {
        height: this.editorRows * 24
      });
    }

    if (this.editorWidth) {
      Ext.apply(editor, {
        width: this.editorWidth
      });
    }

    defaultItems.push(editor, statusBar);
    if (this.layout !== 'hbox') {
      this.msgTarget = 'under';
    }
    this.items = defaultItems;

    this.callParent(arguments);
  },
  getCursorBox: function () {
    return this.getComponent('statusBar').getComponent('cursorPosition');
  },
  getStatus: function () {
    return this.getComponent('statusBar').getComponent('status');
  },
  getExpressionEditor: function () {
    return this.getComponent('expression-editor');
  },
  getAceEditor: function () {
    return this.aceEditor;
  },
  afterRender: function () {
    this.callParent(arguments);

    this.editorEl = this.getExpressionEditor().el;
    this.aceEditor = ace.edit(this.editorEl.id);
    this.aceEditor.infoModelId = this.infoModelId;
    this.aceEditor.component = this;
    this.aceEditor.setOptions({
      enableBasicAutocompletion: true,
      enableLiveAutocompletion: true,
      showLineNumbers: this.showLineNumbers,
      showGutter: this.showGutter,
      fontSize: this.editorFontSize || 12,
      theme: this.DEFAULT_EDITOR_THEME,
      selectionStyle: 'text',
      mode: 'ace/mode/mosaic'
    });
    this.aceEditor.on('blur', Ext.bind(this.onEditorBlur, this));
    this.aceEditor.on('change', Ext.bind(this.onEditorChange, this));
    this.aceEditor.getSelection().on('changeCursor', Ext.bind(this.onCursorChange, this, [this.aceEditor], true));
    this.typeAheadTask = new Ext.util.DelayedTask(this.validateEditor, this);
    this.languageTools.addCompleter(this.expressionCompleter());
    this.updateLayout();
    this.setValue(this.value);
  },
  hasRecord: function () {
    return !!(this.getParentSectionView() && this.getParentSectionView().getRecord());
  },
  getRecord: function () {
    if (this.hasRecord()) {
      return this.getParentSectionView().getRecord();
    }
  },
  getExpressionStore: function () {
    if (this.hasRecord()) {
      return this.getParentSectionView().getRecord().mdExpressionStore;
    }
  },
  getRawValue: function () {
    return this.getAceEditor().getValue();
  },
  getValue: function () {
    return this.value;
  },
  setRawValue: function (value, cursorPosition) {
    if (this.aceEditor) {
      this.getAceEditor().setValue(value, cursorPosition ? 1 : -1);

      if (cursorPosition) {
        this.getAceEditor().moveCursorToPosition(cursorPosition);
      }
    }
  },
  setValue: function (value, cursorPosition) {
    if (this.hasRecord()) { // form field
      var findValue;

      if (Ext.isArray(value)) { // there are expression nodes

        this.updatePurposeCode(value);

        findValue = _.find(value, function (node) {
          return node.CompiledLogic && !node.ParentExpressionID;
        });

        if (findValue) {
          this.rawValue = findValue.CompiledLogic;
        }
      } else if (value) { // initial load might be an ID value because this field is essentially a place holder
        findValue = this.getExpressionStore().findBy(function (record) {
          return record.get('CompiledLogic') && !record.get('ParentExpressionID');
        });

        if (findValue !== -1) {
          this.rawValue = this.getExpressionStore().getAt(findValue).get('CompiledLogic');
        }
      } else { // at this point the user has removed the rule/expression
        this.rawValue = '';
      }
    } else { // field in the form builder
      this.rawValue = Ext.isArray(value) ? this.parseValue(value) : '';
    }

    this.setRawValue(this.rawValue, cursorPosition);
    this.value = value;
    this.resolveExpressionProperties(value);

    if (this.purposeCode === 'VA' || this.purposeCode === 'CS') {
      var selected = this.previousSibling('grid').getSelectionModel().getSelection();

      if (selected.length) {
        switch (this.purposeCode) {
          case 'VA':
            selected[0].set({
              validationExpression: this.rawValue,
              validationRule: this.value
            });
            break;
          case 'CS':
            selected[0].set({
              displayExpression: this.rawValue,
              whenExpression: this.value
            });
            break;
        }
      }
    }
  },
  // For data loss warning (Context Sensitivity)
  setFieldValue: function (value) {
    if (this.hasRecord()) {
      var aceLine = this.el.select('.ace_line');

      // for some reason the actual value of the editor is cleared on hide,
      // however, the the html element does not update so force the update.
      if (aceLine) {
        aceLine.setHTML(value);
      }
      this.setValue(value, 1);
      this.getCursorBox().update('0:0');

      if (!value) {
        this.getExpressionStore().removeAll();
      }
    }
  },
  parseValue: function (expression) {
    var expressionString = '';

    if (Ext.isArray(expression) && expression.length) {
      var infoModel = MC.app.getInfoModel(this.infoModelId),
          compiledLogic = _.find(expression, function (node) { return !!node.CompiledLogic; }).CompiledLogic;

      expressionString = Expressions.parseCompiledLogic(compiledLogic);

      _.each(expression, function (node) {
        if (node.FieldID || node.ExpressionFieldID) {
          var fieldName = infoModel.getField(node.FieldID || node.ExpressionFieldID, 'Name');

          expressionString = expressionString.replace(node.FieldID || node.ExpressionFieldID, fieldName);
        }
      });
    } else {
      return expression;
    }

    return expressionString;
  },
  compileValue: function (expression) {
    if (this.infoModelId) {
      var infoModel = MC.app.getInfoModel(this.infoModelId),
          field = infoModel.getField(this.fieldModelId),
          compiledLogic = '|*"' + this.infoModelId + '":"' + field.VIEW_ID + '"*|\n' + this.getRawValue();

      _.each(expression, function (node) {
        if (node.FieldID || node.ExpressionFieldID) {
          var fieldName = infoModel.getField(node.FieldID || node.ExpressionFieldID, 'Name');

          compiledLogic = compiledLogic.replace(fieldName, node.FieldID);
        }
      });

      return compiledLogic;
    } else {
      return this.getRawValue();
    }
  },
  // TODO: maybe change the validate api call to return correct property names so we don't have to do this.
  // TODO: if we make the change it will most likely break legacy code.
  resolveExpressionProperties: function (expression) {
    var infoModel = MC.app.getInfoModel(this.infoModelId);

    _.each(expression, function (node) {
      if (node.FieldID) {
        Ext.apply(node, {
          ExpressionFieldID: node.FieldID,
          ExpressionFieldName: infoModel.getField(node.FieldID, 'Name')
        });
        delete node.FieldID;
      }

      if (node.LiteralValue) {
        node.ExpressionLiteralValue = node.LiteralValue;
        delete node.LiteralValue;
      }

      if (node.FunctionID) {
        node.InvokedFunctionID = node.FunctionID;
        delete node.FunctionID;
      }

      if (!Ext.isEmpty(node.SequenceOrder)) {
        node.ExpressionSequence = node.SequenceOrder.toString();
        delete node.SequenceOrder;
      }
    }, this);
  },
  expressionCompleter: function () {
    var fieldMap = [],
        keywordsMap = _.map(MC.app.expressionFunctions.data.getRange(), function (record) {
          if (record.data.IsOperator) {
            return {
              name: record.data.codeName,
              value: record.data.codeName,
              meta: 'operator'
            };
          } else   {
            return {
              name: record.data.codeName,
              value: record.data.codeName,
              meta: '\u0192' + '(x)'
            };
          }
        }),
        personalOptionsMap = _.map(MC.app.getPersonalOptionVariables(), function (variable) {
          return {
            name: '$' + variable,
            value: '$' + variable,
            meta: 'option'
          };
        });

    if (this.infoModelId) {
      var infoModel = MC.app.getInfoModel(this.infoModelId);

      fieldMap = _.map(infoModel.getFieldMemo,function (field) {
        return {
          name: infoModel.concatViewAndField(field),
          value: infoModel.concatViewAndField(field),
          meta: 'field'
        };
      });
    }

    return {
      getCompletions: function (editor, session, pos, prefix, callback) {
        callback(null, Ext.Array.merge(fieldMap, keywordsMap, ExpressionEditor.prototype.EXPRESSION_VARIABLES, personalOptionsMap));
      }
    };
  },
  onEditorBlur: function (metaData, editor) {
    // The value of the editor is in a child view (mdExpression)
    if (this.hasRecord() && this.getExpressionStore()) {
      this.getExpressionStore().removeAll();
      this.getExpressionStore().add(this.getValue());
      this.getRecord().dirty = true; // the record must be marked as dirty so the writer can "ADD" the child views of mdExpression
    }
  },
  onEditorChange: function (metaData, editor) {
    this.typeAheadTask.delay(1000);
  },
  onCursorChange: function (e, selection, editor) {
    var cursorPosition = editor.getCursorPosition();

    this.getCursorBox().el.setHTML((cursorPosition.row + 1) + ':' + cursorPosition.column);
  },
  validateEditor: function () {
    if (this.getRawValue() && this.isDirty()) {
      this.validate();
    } else if (!this.getRawValue() && this.getValue()) {
      this.setValue('');
    }
  },
  validate: function () {
    if (this.validationSession) {
      this.validationSession.validate(this.getRawValue(), Ext.bind(this.validationCallback, this));
    } else {
      this.validationSession = new MC.builder.Expression(this.infoModelId, this.getRawValue(), Ext.bind(this.validationCallback, this));
    }
  },
  validationCallback: function (json) {
    var errors = json.errors, marker, parentExpressionNode;

    this.removeMarkers();
    if (errors.length) {
      var errorsTpl = _.map(errors, function (error) {
        var Range = ace.require("ace/range").Range,
            range = new Range(error.startLine - 1, error.startColumn, error.endLine - 1, error.endColumn),
            errorMsg = Ext.htmlEncode(error.problem + ' at position ' + error.endColumn + ' on line ' + error.endLine),
            errorRow = range.end.row;

        marker = this.aceEditor.session.addMarker(range, "invalid-expression", "text", false);
        this.aceEditor.session.addGutterDecoration(errorRow, 'expression-error');
        this.markerIds.push(marker);
        this.gutterDecor.push(errorRow);
        Ext.defer(function (errorRow) {
          this['expressionError' + errorRow] = Ext.create('Ext.tip.ToolTip', {
            target: Ext.get(Ext.DomQuery.select('.expression-error')[0]),
            trackMouse: true,
            html: errorMsg,
            id: Ext.id(null, 'expression-error-'),
            dismissDelay: 15000
          });
        }, 500, this, [errorRow]);

        return errorMsg;
      }, this);
      this.updateStatus(errorsTpl, 'error');
      this.errors = errorsTpl;
    } else {
      this.clearStatus('error');
      this.removeMarkers();
      this.errors = [];
      parentExpressionNode = _.find(json.mdExpressions, function (node) { return !node.ParentExpressionID; });

      if (parentExpressionNode) {
        parentExpressionNode.CompiledLogic = this.compileValue(json.mdExpressions);
      }

      if (this.purposeCode === 'VA' && !Expressions.isBooleanExpression(json.mdExpressions)) {
        this.updateStatus('Validation expressions must return a boolean(True or False) value.', 'error');
        return;
      }

      if (this.isDirty()) {
        this.setValue(json.mdExpressions, this.getAceEditor().getCursorPosition());
      }
    }
  },
  isDirty: function () {
    return this.getRawValue() !== this.parseValue(this.value);
  },
  getErrors: function () {
    return this.errors || [];
  },
  removeMarkers: function () {
    if (this.markerIds.length) {
      _.each(this.markerIds, function (markerId) {
        this.aceEditor.session.removeMarker(markerId);
      }, this);
    }

    if (this.gutterDecor.length) {
      _.each(this.gutterDecor, function (row) {
        this.aceEditor.session.removeGutterDecoration(row, 'expression-error');

        if (this['expressionError' + row]) {
          this['expressionError' + row].destroy();
        }
      }, this);
    }
  },
  // @param {String} type: error, warning, info
  updateStatus: function (text, type) {
    this.getStatus().el.setHTML(text);

    if (type) {
      this.getComponent('statusBar').el.addCls(['expression-status', 'expression-' + type + '-status']);
    }

    this.getComponent('statusBar').updateLayout();
  },
  clearStatus: function (type) {
    this.getStatus().el.setHTML('');

    if (type) {
      this.getComponent('statusBar').el.removeCls(['expression-status', 'expression-' + type + '-status']);
    }
    this.getComponent('statusBar').updateLayout();
  },
  markInvalid: function (errors) {
    var me = this,
        oldMsg = me.getActiveError(),
        active;

    me.setActiveErrors(Ext.Array.from(errors));
    active = me.getActiveError();
    if (oldMsg !== active) {
      me.setError(active);
    }
  },
  clearInvalid: function () {
    var me = this,
        hadError = me.hasActiveError();

    delete me.needsValidateOnEnable;
    me.unsetActiveError();
    if (hadError) {
      me.setError('');
    }
  },
  updatePurposeCode: function(expressionsArray) {
    if (this.purposeCode) {
      _.each(expressionsArray, function (expr) {
        expr.PurposeCode = this.purposeCode;
      }, this);
    }
  },
  setError: function (active) {
    var me = this,
        msgTarget = me.msgTarget,
        prop;

    if (me.rendered) {
      if (msgTarget == 'title' || msgTarget == 'qtip') {
        if (me.rendered) {
          prop = msgTarget == 'qtip' ? 'data-errorqtip' : 'title';
        }
        me.getActionEl().dom.setAttribute(prop, active || '');
      } else {
        me.updateLayout();
      }
    }
  },
  onDestroy: function () {
    if (this.aceEditor) {
      this.aceEditor.destroy();
    }
    if (this.validationSession) {
      this.validationSession.endSession();
    }
    this.callParent(arguments);
  },
  focus: function () {
    this.aceEditor.focus();
  },
  blur: function () {
    this.aceEditor.blur();
  },
  disable: function () {
    this.aceEditor.setReadOnly(true);
  },
  enable: function () {
    this.aceEditor.setReadOnly(false);
  },
  setReadOnly: function (readOnly) {
    this.aceEditor.setReadOnly(readOnly);
  },
  runCalculation: function (eventType) {
    Ext.form.field.Base.prototype.runCalculation.call(this, eventType);
  },
  setContextSensitivity: function (eventType) {
    Ext.form.field.Base.prototype.setContextSensitivity.call(this, eventType);
  },
  runValidation: function (eventType) {
    Ext.form.field.Base.prototype.runValidation.call(this, eventType);
  },
  shouldEvaluate: function (eventType, field) {
    Ext.form.field.Base.prototype.shouldEvaluate.call(this, eventType, field);
  },
  hasValue: function () {
    return !!this.getValue();
  }
});
