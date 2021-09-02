<template>
  <div class="smetaForm">
    <RnToolbar ref="mainTableToolbar" />
    <EntityTableForm
        v-if="nodePath.length > 0"
        :treeId="treeId"
        :nodePath="nodePath"
        :entityController="entityController"
        :visualTreeController="visualTreeController"
        :fileController="fileController"
        :entity-type-picker="entityTypePicker"
        :isHideToolbar="false"
        :isHideStatusbar="false"
        :workflowService="workflowService"
        :cascade-mode="2"
        :cascade-max-depth="4"
        :cascade="true"
        :edit-mode="EditMode.InlineEditing"
        @rowClick="rowclick"
        style="--viewport-height: 100%;--toolbar-height: 0px;"
        ref="mainTable"
    />
  </div>
</template>
<script lang="ts">
import {Component, Prop, Vue, Watch, Emit,} from "vue-property-decorator";

// импорты компонентов из РН-СТРИМ
// табличный компонент EntityTableForm.vue
import {EntityTableForm } from "@rnstream/tableentityform";
import {EditMode} from '@rnstream/tableentityform/src/data/enums/EditMode';

import {
  FileController,
  IEntityEditor,
  IEntityTypePicker,
  IWorkflowService,
  RnToolbar,
  RnToolbarGroup,
  VisualTreeNodePathModel,
  VisualTreeNodeQueryModel,
  VisualTreeController,
  EntityConnectionValue,
  RnEntityEditorWrapper,
  EntityMasterModel,

} from "@rnstream/ui";


import {EntityController} from "@/controllers/EntityController";
import {AxiosResponse} from "axios";
import Clonning from "@rnstream/ui/src/helpers/Clonning";

// регистрация компонентов
@Component({
  name: "TableWithChilds",
  components: {
    RnToolbar,
    EntityTableForm,
    RnEntityEditorWrapper
  }
})
// Основной класс модуля TableWithChilds
export default class TableWithChilds extends Vue {

  // Id объекта\вкладки\класса РН-СТРИМ
  @Prop({ type: Number }) treeId: number;

  // Путь до конкректного элемента/папки дерева в левой части РН-СТРИМ
  @Prop() nodePath: VisualTreeNodePathModel[];

  // Контроллер для доступа к визуальным деревьям и методам взаимодействия с ними
  @Prop() visualTreeController: VisualTreeController;

  // Контроллер для доcтупа к сущностям и методам взаимодействия с ними
  @Prop() entityController: EntityController;

  // Контроллер для доступа к файлам
  @Prop() fileController: FileController;

  // Интерфейс IEntityTypePicker
  @Prop() entityTypePicker: IEntityTypePicker;

  // Интерфейс IEntityEditor
  @Prop() entityEditor: IEntityEditor;

  // Интерфейс IWorkflowService
  @Prop() workflowService: IWorkflowService;

  // Признак видимости тулбара таблицы
  @Prop() isHideToolbar: boolean;
  @Prop() hiddenBaseToolbar : boolean

  public entityMasterModel: EntityMasterModel = null;
  public conValues : EntityConnectionValue[] = []
  private EditMode = EditMode


  // триггер скрытия таблиц
  public isHideTable: boolean = true;

  // массив для хранения пути для первой вложенной папки (Коэффициенты)
  public firstChildPath: VisualTreeNodePathModel[] = new Array<VisualTreeNodePathModel>();


  // массив для хранения пути для второй вложенной папки (Разделы ТОС)
  public secondChildPath: VisualTreeNodePathModel[] = new Array<VisualTreeNodePathModel>();




  async mountToolbar(mainTableToolbarGroup :RnToolbarGroup ){
    (this.$refs.mainTable as any).generateToolbarButtons(mainTableToolbarGroup);
  }

  async rowclick(entityData: any): Promise<void> {

    if(entityData.Depth > 1){
      this.$emit("rowClick", entityData);
    }
    else{
      this.$emit("rowClickDown", entityData)
    }

    console.log(entityData, "Обьект строки")

  }

  // метод получения текущего пути выбранной строки
  getAppendedNodePath(
      nodeId: number,
      entityId: number,

  ): VisualTreeNodePathModel[] {
    let localNodePath = this.$deepCopy(this.nodePath) as VisualTreeNodePathModel[];
    //let localNodePath: VisualTreeNodePathModel[] = Clonning.deepCopy(nodePath);
    localNodePath.push(
        new VisualTreeNodePathModel(nodeId, entityId, null, null, null)
    );
    return localNodePath;
  }
  // Получаем элементы для отображения в таблице
  async getElements(path: VisualTreeNodePathModel[]) {
    let dataGrid = await this.visualTreeController.GetNodes(
        new VisualTreeNodeQueryModel({
          TreeId: this.treeId,
          NodeId: null,
          Parameters: { Nodes: path } as any
        })
    );
    return dataGrid.data.Data;
  }

  //---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  // КОНЕЦ ОБРАБОТКИ СОБЫТИЯ rowclick
  //---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
}
</script>
<style scoped>

body {
  margin: 0;
  overflow: hidden;
}

.multiTable {
  border: 1px solid #ccc;
}

.split.split-horizontal {
  overflow-y: auto;
  overflow-x: auto;
}

.split {
  overflow-y: auto;
  overflow-x: auto;
}

::v-deep .v-data-table .v-data-table__wrapper {
  overflow-y: auto !important;
  overflow-x: auto !important;
}
</style>
