<template>
  <div class="smetaForm">
    <div class="multiTable">
      <div class="toolbarTabs">
        <slot name="CustomToolbar">
          <RnToolbar ref="mainTableToolbar" v-show="this.hiddenBaseToolbar"/> <!-- <RnToolbar ref="slotTableToolbar" />-->
        </slot>
        <!-- TODO addOrUpdate ошибка здесь выскакивает, поскольку слот переоределяет тулбар,
             TODO в который мы передавали экшны внутри трехтаблички-->
        <!--        <RnToolbar ref="mainTableToolbar" />-->
      </div>
      <EntityTableForm
          v-if="nodePath.length > 0"
          :treeId="treeId"
          :nodePath="nodePath"
          :entityController="entityController"
          :visualTreeController="visualTreeController"
          :fileController="fileController"
          :isHideToolbar="false"
          :isHideStatusbar="true"
          :workflowService="workflowService"
          :cascadeMode="3"
          :cascade="true"
          @rowClick="rowclick"
          style="--viewport-height: 100%;--toolbar-height: 0px;"
          ref="mainTable"
      />

    </div>




  </div>
</template>
<script lang="ts">
import {Component, Prop, Vue, Watch, Emit,} from "vue-property-decorator";

// импорты компонентов из РН-СТРИМ
// табличный компонент EntityTableForm.vue
import {EntityTableForm } from "@rnstream/tableentityform";
// импорты табличного компонента с вкладками и карточкой (TabView) и измененного EntityTableForm с логикой даблклика(TableWithCard)
import {TabView, TabViewModel} from "@rnstream/complextableform";


import {
  DisplayTypeEnum,
  EntityEditorData,
  EntityTypeNodeModel,
  FileController,
  IEntityEditor,
  IEntityTypePicker,
  IWorkflowService,
  RnToolbar,
  RnToolbarGroup,
  VisualTreeNodePathModel,
  VisualTreeNodeQueryModel,
  VisualTreeNodeQueryParameter,
  VisualTreeController,
  EntityConnectionValue,
  RnEntityEditorWrapper,
  EntityEditorInfoObject,
  EntityMasterModel,
  EntityModel,
  PageModel
} from "@rnstream/ui";


import {EntityController} from "@/controllers/EntityController";
import {AxiosResponse} from "axios";
import Clonning from "@rnstream/ui/src/helpers/Clonning";

// регистрация компонентов
@Component({
  name: "TableWithChilds",
  components: {
    RnToolbar,
    TabView,
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

  // табличная модель
  @Prop() data: TabViewModel;

  @Prop() hiddenBaseToolbar : boolean

  public entityMasterModel: EntityMasterModel = null;
  public conValues : EntityConnectionValue[] = []



  // триггер скрытия таблиц
  public isHideTable: boolean = true;

  // массив для хранения пути для первой вложенной папки (Коэффициенты)
  public firstChildPath: VisualTreeNodePathModel[] = new Array<VisualTreeNodePathModel>();

  // табличная модель для первой вложенной папки (Коэффициенты)
  public firstChildData: TabViewModel;

  // массив для хранения пути для второй вложенной папки (Разделы ТОС)
  public secondChildPath: VisualTreeNodePathModel[] = new Array<VisualTreeNodePathModel>();

  // табличная модель для второй вложенной папки (Разделы ТОС)
  public secondChildData: TabViewModel;




  async rowclick(entityData: any): Promise<void> {

    // зануляем результаты предыдущего клика

    //получить текущий путь кликнутой строки
    const entityNodePath = this.getAppendedNodePath(
        entityData.NodeId,
        Number(entityData.EntityId),
        this.nodePath
    );

    this.$emit("rowClick", entityNodePath);
    this.$emit("rowClickDown", entityData)
    //получить данные кликнутой строки
    const clickedEntityEditorData = new EntityEditorData(
        Number(entityData.EntityTypeId),
        Number(entityData.EntityId),
        entityNodePath
    );
  }

  // метод получения текущего пути выбранной строки
  getAppendedNodePath(
      nodeId: number,
      entityId: number,
      nodePath: VisualTreeNodePathModel[]
  ): VisualTreeNodePathModel[] {
    let localNodePath = this.$deepCopy(nodePath) as VisualTreeNodePathModel[];
    //let localNodePath: VisualTreeNodePathModel[] = Clonning.deepCopy(nodePath);
    let parentNodeId = localNodePath[localNodePath.length - 1].NodeId;
    localNodePath.push(
        new VisualTreeNodePathModel(nodeId, entityId, parentNodeId, null, null)
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
